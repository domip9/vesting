// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

/**
 * @title VestingContract
 * @dev This contract handles the vesting of a ERC20 tokens for a given beneficiary. Custody of the _token amount
 * can be given to this contract, which will release the _token to the beneficiary following a given schedule.
 * The vesting schedule is established by an array of basis point. Each index of that array
 * corresponds to a month after the vesting period starts (aka after the locking period is over)
 */
contract VestingContract is Ownable {
    // Token
    using SafeERC20 for IERC20;
    IERC20 immutable LEGACY_TOKEN;

    // Structs
    struct Investor {
        uint256 tokens_received; // amount of tokens the Investor has received
        uint256 tokens_promised; // amount of tokens the Investor is owed
        uint256 vesting_start; // saves the date in which the locking/cliff period ends
        //and the vesting (token releases) starts
    }

    // Mappings
    mapping(address => Investor) public walletToInvestor;

    //Variables
    address[] public investorAddresses; // array with all the investor aaddresses
    uint256[] public percentPerMilestone; //percentages per month that will be released (in basis points)
    uint256 public erc20Released; // total amount of released tokens
    uint256 public totalTokensVested;
    uint256 constant SECONDS_IN_MONTH = 30 days;
    address public immutable TOKEN_ADDRESS; // token address
    bool internal locked; // boolean to prevent reentrancy

    // Events
    event ERC20Released(address indexed _token, uint256 amount);

    // Errors
    error contractLacksFunds(uint256 amountNeeded, uint256 contract_balance);
    error addressAmountMismatch(uint256 numAmounts, uint256 numAddresses);
    error LockOrCliffNotOver(uint256 currentTime, uint256 vestingStart);
    error inputTooLarge(uint8 intendedVal, uint256 val);
    error addressAlreadyVested(address investorAddress);
    error invalidAddress();
    error percSumIncorrect(uint256 percSum);
    error allTokensClaimed();
    error notInvestor();

    // Modifiers
    /**
     * @dev Prevents reentrancy
     */
    modifier noReentrant() {
        require(!locked, "No re-entrancy");
        locked = true;
        _;
        locked = false;
    }

    /**
     * @dev Only investors can call functions
     */
    modifier onlyInvestor() {
        if (walletToInvestor[msg.sender].tokens_promised == 0) {
            revert notInvestor();
        }
        _;
    }

    // Constructor
    constructor(
        uint256[]
            memory percentPerMilestone_ /* has the basis point values of the percentages
                                                of tokens that should be released in total in
                                                the indexed month
                                                */,
        address token_address_ // address of the token to be used
    ) Ownable(msg.sender) {
        // check if the percentages add up to 100%
        uint256 percSum = sumArr(percentPerMilestone_);
        if (percSum != 10_000) {
            revert percSumIncorrect(percSum);
        }

        // establish a limit of 10 years for vesting period
        if (percentPerMilestone_.length > 120) {
            revert inputTooLarge(120, percentPerMilestone_.length);
        }

        // verify it is not the zero address (avoid locked funds)
        if (token_address_ == address(0)) {
            revert invalidAddress();
        }

        // percentage of tokens an investor can withdraw at each milestone
        percentPerMilestone = percentPerMilestone_;

        // establish token address
        TOKEN_ADDRESS = token_address_;

        // establish the IERC20 for legacy token to contract interactions
        LEGACY_TOKEN = IERC20(TOKEN_ADDRESS);

        // Initialize the reentrancy variable to not locked
        locked = false;
    }

    // Functions
    /**
     * @dev Set the multiple investors and their promised tokens
     */
    function newMulInvestors(
        address[] memory addressesO50I, // array with the addresses of the investors with 50k to 250ks
        address[] memory addressesO250I, // ... more than 250ks
        uint256[]
            memory tokensO50I /* array with the token amount to be to the equivalent
         indexed address in addressesO50I */,
        uint256[] memory tokensO250I, // ... addres in addressesO250I
        uint256 timeLockO250I // ... more than 250ks
    ) public onlyOwner {
        // check if the contract has funds for the multiple users the owner wants to add
        uint256 amountNeeded = sumArr(tokensO50I) + sumArr(tokensO250I);

        if (
            LEGACY_TOKEN.balanceOf(address(this)) - totalTokensVested <
            amountNeeded
        ) {
            revert contractLacksFunds(
                amountNeeded,
                LEGACY_TOKEN.balanceOf(address(this)) - totalTokensVested
            );
        }

        // number of investors with...
        uint256 numO50I = addressesO50I.length; // with 50k-250ks
        uint256 numO250I = addressesO250I.length; // more than 250ks

        // require that the  number of addresses is equal to the token amounts
        if (numO50I != tokensO50I.length) {
            revert addressAmountMismatch(numO50I, tokensO50I.length);
        }

        if (numO250I != tokensO250I.length) {
            revert addressAmountMismatch(numO250I, tokensO250I.length);
        }

        // require that the loop is capped at 180 new investors
        if (tokensO50I.length + tokensO250I.length > 180) {
            revert inputTooLarge(180, tokensO50I.length + tokensO250I.length);
        }

        // create an Investor struct for each investor with less than 50ks
        for (uint256 i = 0; i < numO50I; ++i) {
            // get the amount of tokens and the address correpsonding to this investor
            address investorAddress = addressesO50I[i];
            uint256 investorTokens = tokensO50I[i];

            // the investor should not already by locked
            if (walletToInvestor[investorAddress].tokens_promised != 0) {
                revert addressAlreadyVested(investorAddress);
            }

            // verify it is not the zero address of this contracts address (avoid locked funds)
            if (
                investorAddress == address(0) ||
                investorAddress == address(this)
            ) {
                revert invalidAddress();
            }

            // create the new Investor struct
            Investor memory investor = Investor(
                0,
                investorTokens,
                block.timestamp
            );

            // map the new Investor address to its struct
            walletToInvestor[investorAddress] = investor;

            // add to the investor array of addresses
            investorAddresses.push(investorAddress);
        }

        // create an Investor struct for each investor with more than 250ks
        for (uint256 i = 0; i < numO250I; ++i) {
            // get the amount of tokens and the address correpsonding to this investor
            address investorAddress = addressesO250I[i];
            uint256 investorTokens = tokensO250I[i];

            // the investor should not already by locked
            if (walletToInvestor[investorAddress].tokens_promised != 0) {
                revert addressAlreadyVested(investorAddress);
            }

            // verify it is not the zero address of this contracts address (avoid locked funds)
            if (
                investorAddress == address(0) ||
                investorAddress == address(this)
            ) {
                revert invalidAddress();
            }

            // create the new Investor struct
            Investor memory investor = Investor(
                0,
                investorTokens,
                block.timestamp + timeLockO250I
            );

            // map the new Investor address to its struct
            walletToInvestor[investorAddress] = investor;

            // add to the investor array of addresses
            investorAddresses.push(investorAddress);
        }

        // update the amount of tokens vested
        totalTokensVested = totalTokensVested + amountNeeded;
    }

    /**
     * @dev Release the tokens
     *
     * Emits a {ERC20Released} event.
     */
    function release() public virtual noReentrant onlyInvestor {
        // gets the amount of tokens the investor can withdraw
        uint256 releasable = canReleasePercent(msg.sender);

        // updates the investors struct to reflect the withdraw
        walletToInvestor[msg.sender].tokens_received =
            walletToInvestor[msg.sender].tokens_received +
            releasable;

        // transfers the tokens to the investor
        LEGACY_TOKEN.safeTransfer(msg.sender, releasable);

        // adds that amount to the total released amount
        erc20Released = erc20Released + releasable;

        // update the amount of tokens vested
        totalTokensVested = totalTokensVested - releasable;

        // emits an event
        emit ERC20Released(msg.sender, releasable);
    }

    /**
     * @dev Adds a new Investor.
     */
    function newInvestor(
        uint256 amount,
        uint256 timeLock,
        address newInvestorAddress
    ) public onlyOwner {
        // the investor should not already be vested
        if (walletToInvestor[newInvestorAddress].tokens_promised != 0) {
            revert addressAlreadyVested(newInvestorAddress);
        }

        // check if the contract has funds for the new user the owner wants to add
        if (
            LEGACY_TOKEN.balanceOf(address(this)) - totalTokensVested < amount
        ) {
            revert contractLacksFunds(
                amount,
                LEGACY_TOKEN.balanceOf(address(this)) - totalTokensVested
            );
        }
        // create the new Investor struc
        Investor memory investor = Investor(
            0,
            amount,
            block.timestamp + timeLock
        );

        // push the new Investor struct to the Investors array
        walletToInvestor[newInvestorAddress] = investor;

        // add to the investor array of addresses
        investorAddresses.push(newInvestorAddress);

        // update the amount of tokens vested
        totalTokensVested = totalTokensVested + amount;
    }

    /**
     * @dev Calculates and returns the amount of tokens that can be withdrawn,
     * as function of milestones passed.
     */
    function canReleasePercent(
        address callerAddress
    ) public view returns (uint256) {
        // get the quatity of tokens he has already withdrawned,
        uint256 hasWithdrawn = walletToInvestor[callerAddress].tokens_received;
        // gets the investors promised tokens
        uint256 tokensPromised = walletToInvestor[callerAddress]
            .tokens_promised;

        // require that the investor has not already withdrawn everything
        if (hasWithdrawn >= tokensPromised) {
            revert allTokensClaimed();
        }

        // gets the current time
        uint256 currentTime = block.timestamp;
        // gets the time in which the investors vesting starts
        uint256 vestingStart = walletToInvestor[callerAddress].vesting_start;

        /* require that the first month of cliff has passed (and as consequence, checks if the lock
        period has passed already)*/
        if (currentTime < vestingStart) {
            revert LockOrCliffNotOver(currentTime, vestingStart);
        }

        // will save the amount that is calculated that can be released
        uint256 canRelease = 0;
        // saves the seconds the investor has spent vesting in each bracket, to allow the token withdraw calc
        uint256 secondsInBracket = 0;

        //Loop through every month until we reach the one the  vesting of this investor is currently on
        for (uint256 i = 1; i <= percentPerMilestone.length; ++i) {
            // if the time the investor has been vesting is bigger than i month
            if (currentTime > (vestingStart + SECONDS_IN_MONTH * i)) {
                // for each bracket, claculate the the tokens to be released in that period
                canRelease =
                    canRelease +
                    (tokensPromised * percentPerMilestone[i - 1]) /
                    10_000; // remember the percent_per
                // milestone are percents multiplied by 10_000
            } else {
                //If the current time is in the middle of one of the months
                secondsInBracket =
                    currentTime -
                    SECONDS_IN_MONTH *
                    (i - 1) -
                    vestingStart;

                canRelease =
                    canRelease +
                    (tokensPromised *
                        percentPerMilestone[i - 1] *
                        secondsInBracket) /
                    10_000 /
                    SECONDS_IN_MONTH;

                break;
            }
        }

        // get the amount of tokens the investor can actually withdraw:
        return canRelease - hasWithdrawn;
    }

    /**
     * @dev Check if the contract has funds for the multiple users the owner wants to add
     */
    function sumArr(
        uint256[] memory arr
    ) internal pure returns (uint256 result) {
        for (uint256 i = 0; i < arr.length; ++i) {
            result = result + arr[i];
        }

        return result;
    }
}
