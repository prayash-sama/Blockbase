# Blockbase

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

interface IERC20 {
    function transferFrom(address sender, address recipient, uint256 amount) external returns (bool);
    // ADDED: The standard transfer function needed to send refunds back
    function transfer(address recipient, uint256 amount) external returns (bool);
}

contract TokenVoting {
    IERC20 public voteToken;
    address public owner;
    
    bool public votingOpen;
    
    mapping(address => bool) public hasVoted;
    // ADDED: Tracks exactly how many tokens each person committed so we can refund them later
    mapping(address => uint256) public committedTokens;

    struct Candidate {
        string name;
        string description;
        uint256 totalVotes;
    }

    Candidate[] private _candidates;
    uint256 private _winnerId;

    event CandidateAdded(uint256 indexed candidateId, string name);
    event VoteCast(address indexed voter, uint256 indexed candidateId, uint256 weight);
    event VotingClosed(uint256 indexed winnerId, string winnerName, uint256 winnerVotes);
    // ADDED: An event to log when someone successfully claims their refund
    event RefundClaimed(address indexed voter, uint256 amount);

    modifier onlyOwner() {
        require(msg.sender == owner, "TokenVoting: caller is not the owner");
        _;
    }

    modifier onlyWhenOpen() {
        require(votingOpen, "TokenVoting: voting is closed");
        _;
    }

    constructor(address _tokenAddress) {
        voteToken = IERC20(_tokenAddress);
        owner = msg.sender;
        votingOpen = true; 
    }

    function addCandidate(string memory name, string memory description) public onlyOwner onlyWhenOpen {
        _candidates.push(Candidate({
            name: name,
            description: description,
            totalVotes: 0
        }));
        
        emit CandidateAdded(_candidates.length - 1, name);
    }

    function vote(uint256 candidateId, uint256 amount) public onlyWhenOpen {
        require(!hasVoted[msg.sender], "TokenVoting: address has already voted");
        require(candidateId < _candidates.length, "TokenVoting: invalid candidate ID");
        require(amount > 0, "TokenVoting: vote amount must be > 0");

        require(
            voteToken.transferFrom(msg.sender, address(this), amount), 
            "TokenVoting: token transfer failed. Did you approve?"
        );

        hasVoted[msg.sender] = true;
        // ADDED
