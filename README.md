// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

interface IERC20 {
    function transfer(address to, uint256 amount) external returns (bool);
    function balanceOf(address account) external view returns (uint256);
}

contract TokenVesting {
    IERC20 public immutable token;

    struct Schedule {
        uint256 totalAmount;
        uint256 released;
        uint256 start;
        uint256 duration;
    }

    mapping(address => Schedule) public schedules;

    event VestingCreated(
        address indexed beneficiary,
        uint256 amount,
        uint256 start,
        uint256 duration
    );

    event TokensReleased(
        address indexed beneficiary,
        uint256 amount
    );

    constructor(address tokenAddress) {
        require(tokenAddress != address(0), "Invalid token");
        token = IERC20(tokenAddress);
    }

    function createVesting(
        address beneficiary,
        uint256 amount,
        uint256 start,
        uint256 duration
    ) external {
        require(beneficiary != address(0), "Invalid beneficiary");
        require(amount > 0, "Amount is zero");
        require(duration > 0, "Duration is zero");
        require(
            schedules[beneficiary].totalAmount == 0,
            "Schedule exists"
        );

        schedules[beneficiary] = Schedule({
            totalAmount: amount,
            released: 0,
            start: start,
            duration: duration
        });

        emit VestingCreated(
            beneficiary,
            amount,
            start,
            duration
        );
    }

    function vestedAmount(
        address beneficiary
    ) public view returns (uint256) {
        Schedule memory s = schedules[beneficiary];

        if (s.totalAmount == 0 || block.timestamp < s.start) {
            return 0;
        }

        uint256 elapsed = block.timestamp - s.start;

        if (elapsed >= s.duration) {
            return s.totalAmount;
        }

        return (s.totalAmount * elapsed) / s.duration;
    }

    function releasable(
        address beneficiary
    ) public view returns (uint256) {
        uint256 vested = vestedAmount(beneficiary);
        return vested - schedules[beneficiary].released;
    }

    function release() external {
        uint256 amount = releasable(msg.sender);

        require(amount > 0, "Nothing to release");

        schedules[msg.sender].released += amount;

        require(
            token.transfer(msg.sender, amount),
            "Transfer failed"
        );

        emit TokensReleased(msg.sender, amount);
    }
}
