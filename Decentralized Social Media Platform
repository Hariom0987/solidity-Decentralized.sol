// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

interface IERC20 {
    function transfer(address to, uint256 amount) external returns (bool);
    function transferFrom(address from, address to, uint256 amount) external returns (bool);
    function balanceOf(address account) external view returns (uint256);
}

contract DAO {
    struct Proposal {
        uint256 id;
        string title;
        string description;
        address proposer;
        uint256 amount;
        address recipient;
        uint256 votesFor;
        uint256 votesAgainst;
        uint256 startTime;
        uint256 endTime;
        bool executed;
        mapping(address => bool) hasVoted;
        mapping(address => bool) voteChoice; // true = for, false = against
    }
    
    struct Member {
        address memberAddress;
        uint256 votingPower;
        uint256 joinTime;
        bool isActive;
    }
    
    mapping(uint256 => Proposal) public proposals;
    mapping(address => Member) public members;
    mapping(address => uint256) public memberIndex;
    address[] public memberList;
    
    uint256 public proposalCount;
    uint256 public totalVotingPower;
    uint256 public constant VOTING_PERIOD = 7 days;
    uint256 public constant MINIMUM_QUORUM = 51; // 51% participation required
    uint256 public constant MINIMUM_MAJORITY = 60; // 60% approval required
    uint256 public constant PROPOSAL_DEPOSIT = 1 ether;
    
    address public admin;
    bool public isInitialized;
    
    event MemberAdded(address indexed member, uint256 votingPower);
    event MemberRemoved(address indexed member);
    event ProposalCreated(
        uint256 indexed proposalId,
        string title,
        address indexed proposer,
        uint256 amount,
        address indexed recipient
    );
    event VoteCast(
        uint256 indexed proposalId,
        address indexed voter,
        bool support,
        uint256 votingPower
    );
    event ProposalExecuted(uint256 indexed proposalId, bool success);
    event FundsReceived(address indexed from, uint256 amount);
    
    modifier onlyMember() {
        require(members[msg.sender].isActive, "Only active members can perform this action");
        _;
    }
    
    modifier onlyAdmin() {
        require(msg.sender == admin, "Only admin can perform this action");
        _;
    }
    
    modifier proposalExists(uint256 _proposalId) {
        require(_proposalId < proposalCount, "Proposal does not exist");
        _;
    }
    
    modifier votingActive(uint256 _proposalId) {
        Proposal storage proposal = proposals[_proposalId];
        require(block.timestamp >= proposal.startTime, "Voting has not started");
        require(block.timestamp <= proposal.endTime, "Voting period has ended");
        require(!proposal.executed, "Proposal already executed");
        _;
    }
    
    constructor() {
        admin = msg.sender;
        isInitialized = false;
    }
    
    /**
     * @dev Initialize DAO with founding members
     * @param _foundingMembers Array of founding member addresses
     * @param _votingPowers Array of voting powers for each founding member
     */
    function initializeDAO(
        address[] memory _foundingMembers,
        uint256[] memory _votingPowers
    ) external onlyAdmin {
        require(!isInitialized, "DAO already initialized");
        require(_foundingMembers.length == _votingPowers.length, "Arrays length mismatch");
        require(_foundingMembers.length > 0, "Must have at least one founding member");
        
        for (uint256 i = 0; i < _foundingMembers.length; i++) {
            address member = _foundingMembers[i];
            uint256 power = _votingPowers[i];
            
            require(member != address(0), "Invalid member address");
            require(power > 0, "Voting power must be greater than 0");
            require(!members[member].isActive, "Member already exists");
            
            members[member] = Member({
                memberAddress: member,
                votingPower: power,
                joinTime: block.timestamp,
                isActive: true
            });
            
            memberIndex[member] = memberList.length;
            memberList.push(member);
            totalVotingPower += power;
            
            emit MemberAdded(member, power);
        }
        
        isInitialized = true;
    }
    
    /**
     * @dev Core Function 1: Create a new proposal
     * @param _title Title of the proposal
     * @param _description Description of the proposal
     * @param _amount Amount of ETH to transfer (0 for non-financial proposals)
     * @param _recipient Address to receive funds (address(0) for non-financial proposals)
     */
    function createProposal(
        string memory _title,
        string memory _description,
        uint256 _amount,
        address _recipient
    ) external payable onlyMember returns (uint256) {
        require(isInitialized, "DAO not initialized");
        require(bytes(_title).length > 0, "Title cannot be empty");
        require(bytes(_description).length > 0, "Description cannot be empty");
        require(msg.value >= PROPOSAL_DEPOSIT, "Insufficient proposal deposit");
        
        if (_amount > 0) {
            require(_recipient != address(0), "Invalid recipient for financial proposal");
            require(address(this).balance >= _amount + PROPOSAL_DEPOSIT, "Insufficient DAO funds");
        }
        
        uint256 proposalId = proposalCount++;
        Proposal storage newProposal = proposals[proposalId];
        
        newProposal.id = proposalId;
        newProposal.title = _title;
        newProposal.description = _description;
        newProposal.proposer = msg.sender;
        newProposal.amount = _amount;
        newProposal.recipient = _recipient;
        newProposal.startTime = block.timestamp;
        newProposal.endTime = block.timestamp + VOTING_PERIOD;
        newProposal.executed = false;
        
        emit ProposalCreated(proposalId, _title, msg.sender, _amount, _recipient);
        return proposalId;
    }
    
    /**
     * @dev Core Function 2: Cast vote on a proposal
     * @param _proposalId ID of the proposal to vote on
     * @param _support True for supporting, false for opposing
     */
    function vote(uint256 _proposalId, bool _support) 
        external 
        onlyMember 
        proposalExists(_proposalId) 
        votingActive(_proposalId) 
    {
        Proposal storage proposal = proposals[_proposalId];
        require(!proposal.hasVoted[msg.sender], "Already voted on this proposal");
        
        uint256 voterPower = members[msg.sender].votingPower;
        
        proposal.hasVoted[msg.sender] = true;
        proposal.voteChoice[msg.sender] = _support;
        
        if (_support) {
            proposal.votesFor += voterPower;
        } else {
            proposal.votesAgainst += voterPower;
        }
        
        emit VoteCast(_proposalId, msg.sender, _support, voterPower);
    }
    
    /**
     * @dev Core Function 3: Execute a proposal after voting period
     * @param _proposalId ID of the proposal to execute
     */
    function executeProposal(uint256 _proposalId) 
        external 
        proposalExists(_proposalId) 
    {
        Proposal storage proposal = proposals[_proposalId];
        require(block.timestamp > proposal.endTime, "Voting period not ended");
        require(!proposal.executed, "Proposal already executed");
        
        uint256 totalVotes = proposal.votesFor + proposal.votesAgainst;
        uint256 quorumRequired = (totalVotingPower * MINIMUM_QUORUM) / 100;
        uint256 majorityRequired = (totalVotes * MINIMUM_MAJORITY) / 100;
        
        bool quorumMet = totalVotes >= quorumRequired;
        bool majorityApproval = proposal.votesFor >= majorityRequired;
        bool success = false;
        
        proposal.executed = true;
        
        if (quorumMet && majorityApproval) {
            if (proposal.amount > 0 && proposal.recipient != address(0)) {
                // Execute financial proposal
                if (address(this).balance >= proposal.amount) {
                    payable(proposal.recipient).transfer(proposal.amount);
                    success = true;
                }
            } else {
                // Non-financial proposal automatically succeeds if voted in
                success = true;
            }
        }
        
        // Return proposal deposit to proposer
        payable(proposal.proposer).transfer(PROPOSAL_DEPOSIT);
        
        emit ProposalExecuted(_proposalId, success);
    }
    
    // Additional helper functions
    
    /**
     * @dev Add a new member to the DAO (requires proposal and voting)
     * @param _member Address of new member
     * @param _votingPower Voting power to assign
     */
    function addMember(address _member, uint256 _votingPower) external onlyAdmin {
        require(_member != address(0), "Invalid member address");
        require(_votingPower > 0, "Voting power must be greater than 0");
        require(!members[_member].isActive, "Member already exists");
        
        members[_member] = Member({
            memberAddress: _member,
            votingPower: _votingPower,
            joinTime: block.timestamp,
            isActive: true
        });
        
        memberIndex[_member] = memberList.length;
        memberList.push(_member);
        totalVotingPower += _votingPower;
        
        emit MemberAdded(_member, _votingPower);
    }
    
    /**
     * @dev Remove a member from the DAO
     * @param _member Address of member to remove
     */
    function removeMember(address _member) external onlyAdmin {
        require(members[_member].isActive, "Member does not exist");
        
        uint256 memberPower = members[_member].votingPower;
        members[_member].isActive = false;
        totalVotingPower -= memberPower;
        
        // Remove from member list
        uint256 index = memberIndex[_member];
        uint256 lastIndex = memberList.length - 1;
        
        if (index != lastIndex) {
            address lastMember = memberList[lastIndex];
            memberList[index] = lastMember;
            memberIndex[lastMember] = index;
        }
        
        memberList.pop();
        delete memberIndex[_member];
        
        emit MemberRemoved(_member);
    }
    
    /**
     * @dev Get proposal details
     */
    function getProposal(uint256 _proposalId) 
        external 
        view 
        proposalExists(_proposalId) 
        returns (
            string memory title,
            string memory description,
            address proposer,
            uint256 amount,
            address recipient,
            uint256 votesFor,
            uint256 votesAgainst,
            uint256 startTime,
            uint256 endTime,
            bool executed
        ) 
    {
        Proposal storage proposal = proposals[_proposalId];
        return (
            proposal.title,
            proposal.description,
            proposal.proposer,
            proposal.amount,
            proposal.recipient,
            proposal.votesFor,
            proposal.votesAgainst,
            proposal.startTime,
            proposal.endTime,
            proposal.executed
        );
    }
    
    /**
     * @dev Check if address has voted on a proposal
     */
    function hasVoted(uint256 _proposalId, address _voter) 
        external 
        view 
        proposalExists(_proposalId) 
        returns (bool) 
    {
        return proposals[_proposalId].hasVoted[_voter];
    }
    
    /**
     * @dev Get vote choice of a voter for a proposal
     */
    function getVoteChoice(uint256 _proposalId, address _voter) 
        external 
        view 
        proposalExists(_proposalId) 
        returns (bool) 
    {
        require(proposals[_proposalId].hasVoted[_voter], "Voter has not voted");
        return proposals[_proposalId].voteChoice[_voter];
    }
    
    /**
     * @dev Get all members
     */
    function getAllMembers() external view returns (address[] memory) {
        return memberList;
    }
    
    /**
     * @dev Get DAO stats
     */
    function getDAOStats() 
        external 
        view 
        returns (
            uint256 memberCount,
            uint256 totalVotingPower_,
            uint256 proposalCount_,
            uint256 treasuryBalance
        ) 
    {
        return (
            memberList.length,
            totalVotingPower,
            proposalCount,
            address(this).balance
        );
    }
    
    /**
     * @dev Receive ETH donations
     */
    receive() external payable {
        emit FundsReceived(msg.sender, msg.value);
    }
    
    /**
     * @dev Fallback function
     */
    fallback() external payable {
        emit FundsReceived(msg.sender, msg.value);
    }
}
