# RPSLS
### Variable Field
```solidity
pragma solidity >=0.7.0 <0.9.0;

import "./CommitReveal.sol";
import "./TimeUnit.sol";

contract RPS is CommitReveal, TimeUnit {
    /*
    0 - Rock
    1 - Paper
    2 - Scissor
    3 - Spock
    4 - Lizard
    5 - Undecided
    */
    struct Player {
        uint choice;
        address addr;
        bool isCommitted;
    }
    uint private reward = 0;
    Player[2] private players;
    uint private numPlayer = 0;
    uint private numInput = 0;
    uint private numReveal = 0;
    uint private TIME_LIMIT = 1 minutes;
```
`RPS` inherit `CommitReveal`, `TimeUnit` that allow `RPS` to use function from these contract.

`reward` Amount of reward as ether.

`players` Current players state that have `choice`, `addr`, `isCommitted`.

`numPlayer` Number of joined players.

`numInput` Number of chose players.

`numReveal` Number of revealed players.

`TIME_LIMIT` Time limit constant to use when has no player action that allow player withdraw reward.
<hr/>

### Other Function
#### Check player
```solidity
function checkPlayer() public view {
    require(msg.sender == 0xE0f5206BBD039e7b0592d8918820024e2a7437b9 || msg.sender == 0xAb8483F64d9C6d1EcF9b849Ae677dD3315835cb2 || msg.sender == 0x4B20993Bc481177ec7E8f571ceCaE8A9e22C02db || msg.sender == 0x78731D3Ca6b7E34aC0F824c42a7cC18A495cabaB);
}
```
Use to check player address is assigned address.
```solidity
function hashChoiceWithSalt(uint choice, string memory salt) public pure returns (bytes32) {
    require(choice >= 0 && choice <= 5);
    return keccak256(abi.encodePacked(choice, salt));
}
```
Use to hash choice with salt.
#### Check winner and pay
```solidity
function _checkWinnerAndPay() private {
    uint p0Choice = players[0].choice;
    uint p1Choice = players[1].choice;
    address payable account0 = payable(players[0].addr);
    address payable account1 = payable(players[1].addr);

    if (p0Choice == p1Choice) {
        // to split reward
        account0.transfer(reward / 2);
        account1.transfer(reward / 2);
    }
    else if ((p0Choice + 2) % 5 == p1Choice || (p0Choice + 4) % 5 == p1Choice){
        // to pay player[1]
        account0.transfer(reward);
    }
    else if ((p1Choice + 2) % 5 == p0Choice || (p1Choice + 4) % 5 == p0Choice){
        // to pay player[0]
        account1.transfer(reward);    
    }
    emit GameResult(p0Choice, p1Choice);
}
```
Use to check winner and pay winner when 2 player are revealed their choice.
#### Reset Game
```solidity
function _resetGame() private {
    numPlayer = 0;
    numInput = 0;
    numReveal = 0;
    reward = 0;
    emit GameReset();
}
```
Use to reset state of contract when game ended.
<hr/>

### โค้ดที่ป้องกันการ lock เงินไว้ใน contract และโค้ดส่วนที่จัดการกับความล่าช้าที่ผู้เล่นไม่ครบทั้งสองคนเสียที
```solidity
function playerWithdraw() public {
    // require some player
    require(numPlayer > 0);
    // require sender are player
    require(msg.sender == players[0].addr || msg.sender == players[1].addr);
    // require time limit
    require(elapsedMinutes() > TIME_LIMIT);
    uint playerIdx;
    uint otherPlayerIdx;
    if (msg.sender == players[0].addr){
        playerIdx = 0;
        otherPlayerIdx = 1;
    }
    else if(msg.sender == players[1].addr){
        playerIdx = 1;
        otherPlayerIdx = 0;
    }
    
    require(!commits[players[otherPlayerIdx].addr].revealed, "Opponent already revealed");
    
    address payable account = payable(players[playerIdx].addr);
    reward -= 1 ether;
    account.transfer(1 ether);
    emit PlayerWithdrawn(msg.sender);
    
    // check state
    if (numPlayer == 1){
        numPlayer--;
    }
    else if (numPlayer == 2 && numInput == 0){
        numPlayer--;
        if (playerIdx == 0){
            players[0] = players[1];
        }
    }
    else if (numPlayer == 2 && (numInput == 1 || numInput == 2) && numReveal == 0){
        numPlayer--;
        numInput--;
        if (playerIdx == 0){
            players[0] = players[1];
        }
    }
    // Self reveal and time limit, auto win
    else if (numPlayer == 2 && numInput == 2 && numReveal == 1){
        reward -= 1 ether;
        account.transfer(1 ether);
    }
}
```
ในส่วนของโค้ดที่ป้องกันการ lock เงินใน contract ก็จะมีการเช็คเงื่อนไขต่างๆ เช่น จำนวนผู้เล่น เป็นหนึ่งในผู้เล่น เวลานั้นเกินกว่าที่กำหนดไว้ และจะมีการ
ถอนเงินที่ลงไว้ออกมาตามเงื่อนไขต่างๆ โดยเช็คจาก state ของ contract ซึ่งจะเป็นการคืนค่า state แบบต่างๆที่ทำให้ contract สามารถทำงานต่อไปได้ และจะมีกรณีที่เรา reveal ก่อนแล้วฝ่ายตรงข้ามไม่ reveal ในเวลาที่กำหนด เราจะสามารถถอนเงินรางวัลได้ทั้งหมด เหมือนเป็นการชนะบายนั่นเอง

### โค้ดส่วนที่ทำการซ่อน choice และ commit
```solidity
function playerCommitHashChoice(bytes32 hashChoice) public  {
    // require 2 player
    require(numPlayer == 2);
    // require sender are player
    require(msg.sender == players[0].addr || msg.sender == players[1].addr);

    if (msg.sender == players[0].addr){
        require(!players[0].isCommitted, "Player already committed");
        players[0].isCommitted = true;
    }
    else if(msg.sender == players[1].addr){
        require(!players[1].isCommitted, "Player already committed");
        players[1].isCommitted = true;
    }

    commit(getHash(hashChoice));
    numInput++;
    emit PlayerCommittedHashChoice(msg.sender, numInput);
    setStartTime();
}
```
เป็น function ที่ใช้การ commit choice ของผู้เล่น โดยผู้เล่นจะต้องนำ hash จาก function `hashChoiceWithSalt` มาใส่ใน function นี้เพื่อเป็นการ 
commit โดยจะใช้ CommitReveal.sol เช็คเรื่องการ commit ของ player

### โค้ดส่วนทำการ reveal และนำ choice มาตัดสินผู้ชนะ 
```solidity
function playerReveal(uint choice, string memory salt) public {
    require(numPlayer == 2);
    require(numInput == 2);
    require(msg.sender == players[0].addr || msg.sender == players[1].addr);
    require(choice >= 0 && choice <= 5);
    require(players[0].isCommitted && players[1].isCommitted);

    bytes32 hashChoice = hashChoiceWithSalt(choice, salt);
    reveal(hashChoice);
    numReveal++;
    
    if (msg.sender == players[0].addr){
        players[0].choice = choice;
    }
    else if(msg.sender == players[1].addr){
        players[1].choice = choice;
    }
    
    emit PlayerRevealed(msg.sender, numReveal);

    if (numReveal == 2) {
        _checkWinnerAndPay();
        _resetGame();
    }
    setStartTime();
}
```
เมื่อผู้เล่นมีการ reveal ครบสองคนแล้วก็จะมีการเรียก function `_checkWinnerAndPay()` ที่ใช้ในการแจกรางวัลตามที่ผู้เล่นได้เลือก choice มาและมีการ
`_resetGame()` ที่ใช้ในการ reset state ของ contract เพื่อให้ contract สามารถทำงานต่อไปได้
