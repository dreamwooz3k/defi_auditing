# 008 오디팅 실습

## Jp1n9 / DEX(1) / 수수료 오기 문제
### 설명
DreamDex.sol, swap 함수
x*y=k 기반의 amm은 swap을 할 때 마다 x,y의 비율이 달라짐에 따라서 k의 값도 달라지게 된다. 해당 swap함수를 확인해보면 swap을 해준 뒤 k값을 업데이트하는 부분이 빠져있어 풀에 토큰 두 개의 비율에 대한 k값이 안 바뀌는 문제점이 존재하고, 이에 따라 거래소는 swap을 하는 사용자에게 돈을 더 주는 문제점이 존재한다. 이런 문제점은 LP 토큰의 가치를 떨어뜨리기 때문에 거래소의 투자한 사람들이 피해를 보게 된다.

또한 거래소에서 받는 수수료가 0.1%가 아니라 10%로 구현되어 있다.

### 파급력
이 문제는 x*y=k의 알고리즘이 swap을 할 때 적용되지 않는 문제점이기 때문에 심각도는 Informational이라고 생각한다.

추가적으로 수수료 문제도 마찬가지로 Information이라고 생각한다.
### 해결방안

``` _k= TestToken(tokenX).balanceOf(address(this)) * TestToken(tokenY).balanceOf(address(this)); ```

스왑을 진행하고 다음과 같이 k 값을 업데이트 해주면 해결할 수 있다.

수수료 문제는 해당 변수의 값 수정하여 해결할 수 있다.

## Jp1n9 / DEX (2) / approve, transfer 이중 실행 문제
### 설명
DreamDex.sol, removeLiquidity 함수, 158/159 번째 줄
removeLiquidity 즉, 사용자가 넣었던 유동성을 회수할 때 호출되는 함수인데, LP 토큰을 넣어놨던 유동성과 교환하는 과정에서 transfer로 토큰들을 보내주는데 그 과정전에 해주지 않아도되는 approve과정이 들어가있는데 DEX->msg.sender로 돌려받는 토큰 양만큼 approve를 해준다. ERC20 토큰에서 approve는 transferFrom을 허용할 양을 정해주기 위해 사용되는 함수이기 때문에 transfer를 할 때는 approve를 해줄 필요가 없다. 따라서 악의적인 공격자가 풀에 토큰을 넣고 빼는 과정을 반복하며 trasnferFrom 함수를 같이 호출하여 거래소에 모든 토큰을 빼낼 수 있다.
### 파급력
이 취약점의 파급력은 Critical이라고 생각한다. 거래소 풀에 모든 토큰을 빼낼 수 있기 때문이다.
### 해결방안
해당 approve 두 줄을 지우면 해결할 수 있다.

## rkdnjs / DEX(1) / addliquidity argv issue
### 설명
DEX.sol, addLiquidity 함수
함수 인자 중 minimumLPTokenAmount라는 인자가 있는데 uniswap 같은 곳에서는 이 인자의 목적은 풀에 유동성을 공급해주었을 때 LP 토큰을 받는데 이 때 유동성 공급자가 공급한 토큰의 양에 따라 받을 수 있는 LP 토큰의 양은 달라지기 때문에 유동성을 공급할 때 최소한 이 LP 토큰 이상은 받아야 공급할 것이라는 의사표현을 도와주는 목적으로 사용된다. 이 코드에서는 풀에 토큰이 최소한 이 만큼은 보존하라는 의미로 사용되었고, 유동성 공급자는 그만큼 LP 토큰을 적게 받을 뿐더러 풀에 계속 토큰들이 쌓일 것이고 이 토큰들은 되찾을 수 없는 문제점이 존재한다.
### 파급력
파급력은 information이라고 생각한다. 구현의 문제이기 때문에 이 사실을 아는 사용자들은 이용을 안할려고 할 것이다.
### 해결방안
해결 방안은 minimumLPTokenAmount의 사용 목적을 풀에 이만큼의 토큰을 넣었을 때 최소한 이만큼의 LP토큰을 받겠다는 목적으로 조건을 만들어주면된다.

## rkdnjs / DEX(2) / transfer, remove 구현 오류
### 설명
DEX.sol, removeLiquidity, 112/113, 119/120 번째 줄
112/113번째 줄에서 인자로 받은 minimumToken[x,y]amount에 대해서 값 검사를 하지 않고 사용자에게 돌려 줄 토큰의 개수를 계산하는데 사용하고 있다. 이 과정에서 공격자가 원래 돌려받을 토큰보다 훨씬 많은 값을 넣어서 거래소에 모든 토큰을 빼낼 수 있다. 하지만 아쉽게도 119/120번째 줄에 구현 문제로 인한 _transfer 함수의 인자가 tokenX -> 거래소로 위에서 계산한 값을 보내라고 구현이 되어있어 아쉽게도 토큰을 다 빼내지는 못하였지만, transfer 구현이 정상적으로만 되어있었다면 거래소의 모든 돈을 빼낼 수 있었을 것이다. 하지만 반대로 거래소의 유동성을 제공한 사람은 제공했던 토큰을 다시 돌려받지 못하는 문제점이 존재한다.
### 파급력
파급력은 High라고 생각한다. transfer의 구현이 잘못되어있어 계산하는 과정에 사용하는 값에 대한 검증이 없는 취약점이 존재하지만 토큰을 다 빼낼 수 없다는 점과 유동성을 제공한 사용자가 유동성을 다시 돌려받을 수 없다는 러그풀에 가까운 구현 문제들을 고려한 결과이다.
### 해결방안
인자로 받는 값에 대한 검증과 유동성을 제공한 사람에게 유동성을 다시 돌려받을 수 있게 수정을 해야하면 될 것 같다.

## procfs / DEX(1) / lp토큰 전체량 조작
### 설명
Dex.sol, removeLiquidity, 215번째 줄
removeLiquidity 함수에서 공급받은 유동성을 되돌려줄 때 인자에 lp토큰 값을 1개를 받든 2개를 받든 p.amount 값을 0으로 만든다. 이 과정에서 lpsum에 값을 구할 때 p.amount 값을 이용하게 되는데 만약 사용자가 10 lp 토큰이 존재하고 전체 lp토큰개수가 20개라고 가정을하면 1 lp 토큰 만큼의 유동성을 돌려받고 9 lp 토큰 만큼의 유동성을 돌려 받을 때 받는 값은 9/10이 된다는 의미이다. 따라서 받는 돈의 양을 조작할 수 있다.
### 파급력
파급력은 Critical 이라고 생각한다. lp 토큰의 가치를 떨어 뜨릴 수 있고 풀에 있는 토큰을 많이 받아 낼 수 있기 때문이다. 뿐만 아니라 이 거래소는 수수료 값을 풀에 그대로 저장시키는게 아니라 변수를 만들어 따로 관리하고 수수료를 lp토큰의 지분만큼 보내주는데 이 과정에서도 똑같이 영향을 받아 수수료마저도 빼낼 수 있다.
### 해결방안
전체 lp 토큰의 계산 방식 매커니즘을 totalsupply 함수를 이용하는 등의 방식으로 개선하면 해결할 수 있을 것 같다.

## procfs / DEX(2) / lp토큰 가치 계산 문제
### 설명
Dex.sol, removeLiquidity, 196번째 줄
removeLiquidity 함수에서 공급받은 유동성을 되돌려줄 때 돌려주는 계산을 swap할 때 공식을 동일하게 사용하여 실제 가지고 있는 lp 토큰의 가치보다 적은 돈을 돌려 받는 문제 존재.
### 파급력
파급력은 low라고 생각한다. 실제로 이러한 문제점이 존재하게 된다면 금방 이 사실을 알아차릴 수 있을 것이고 그렇게 되면 해당 거래소를 이용하는 사람은 없을 것이기 때문이다.
### 해결방안
``` tokenYAmount = tokenYBalance * lpTokenAmount / lpSum; ``` 계산 방식을 다음과 같이 변경

## Eunyeong2 / DEX / 수수료 할인 문제
### 설명
dex.sol / _swap 함수 / 67번째 줄
swap을 진행할 때 풀에 넣는 코인에 대해서 수수료를 계산하는데 transferfrom을 할 때 계산한 수수료값을 더한 값이 아닌 뺀 값을 받고 있다.
### 해결방안
수수료 값을 빼는 부분을 더하는 것으로 바꾸면 된다. 

## da-yunjin / DEX / x*y=k amm 구현
### 설명
- lp 토큰을 지급량 계산
- lp 토큰을 이용하여 유동성을 되찾을 때 minimumtoken 값 사용처
- swap 시 토큰에 비율에 따른 x*y=k 공식 미적용
### 파급력
- information
### 해결방안
- x*y=k 공식 적용

## Jp1n9 / lending / 구현
### 설명
Lending.sol / repay, 109 / withdraw, 184 / calcInterest, 199
- repay 함수에서 부분 상환을 진행할 때, 상환한 만큼의 ether 지급과 108번째 줄에서 amount를 뺀 값을 갚은 빚에 적용시켜주는데 이 때 갚은 amount를 빼 버려서 실제로 갚아야할 빚이 늘어난 문제점이 존재한다.
- withdraw를 할 때 투자자가 예치한 토큰을 뺄 때 예치한 금액의 0.1%를 추가로 주는 것처럼 보이는데, 실제로는 예치되어있는 값에 0.1를 추가로 빼 이자를 주는 방식이 의도와 다르게 구현되어있는 것 같다.
- calcInterest 함수에서 이자가 고정으로 되어있어 복리로 이자가 붙지 않는다.
### 파급력
information
### 해결방안
- 연산을 할 때 각 의도마다의 연산 오류를 고치면 된다.

## procfs / lending / 이자 지급 방식
### 설명
Lending.sol / _depositEth, 78번째 줄 / calcPrincipleSum, 51번째 줄
deposit을 할 때 마다 넣은 amount 값에 이자가 붙은 amount 값이 더해지는데 원래 있던 d.amount의 값에 대해 이자를 붙이는 것이 아닌, 새로 값을 덮어 쓴다. 그렇기 때문에 원래 deposit한 값은 사라지고 새롭게 넣은 amount에 대하여 값이 저장되는 문제점이 있을 것 같다.
이자를 계산하는 함수에서 복리로 이자를 더하는 것이 아니라 +가 빠져 결국 amount * 0.1로 계산되는 문제점이 존재한다.
### 파급력
Low, information
### 해결 방안
이자는 원래 있던 amount의 대하여 계산하고 amount 값을 더해주면 될 것 같다.

## Eunyeong2 / lending / 무한 대출
### 설명
Lending.sol / borrow
borrow 함수는 대출자의 담보를 확인하고 담보의 50% 미만의 대출금을 빌려주는데, 대출을 하고나서 따로 이 사람이 대출을 했는지에 대한 검사를 하지 않아 한번 넣은 담보로 계속 대출을 할 수 있는 문제점이 발견되었다. 거래소의 존재하는 모든 usdc를 빼올 수 있다.
### 파급력
dex의 모든 usdc를 빼낼 수 있기 때문에 Critical이다.
### 해결 방안
대출을 해주었을 때 담보금에 대한 체크를 하는 체크 루틴을 따로 추가하여 담보금이 재사용되는 것을 방지해야 한다.
aave 같은 경우 이를 막기 위해서 debt 토큰을 따로 만들고 관리하여 debt 토큰의 발행량으로 이를 체크한다.
```// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "../lib/forge-std/src/Test.sol";
import "src/Lending.sol";
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract USDC is ERC20 {
    uint public constant USDC_INITIAL_SUPPLY = 1000 ether;
    constructor() ERC20("USD Coin", "USDC") {
        super._mint(msg.sender, USDC_INITIAL_SUPPLY);
    }
    function mint(address to, uint amount) public
    {
        _mint(to, amount);
    }
}

contract ETH is ERC20 {
    uint public constant USDC_INITIAL_SUPPLY = 1000 ether;
    constructor() ERC20("ETH coin", "ETH") {
        super._mint(msg.sender, USDC_INITIAL_SUPPLY);
    }

    function mint(address to, uint amount) public
    {
        _mint(to, amount);
    }
}

contract LendingTest is Test {

    MyLend bank;
    DreamOracle oracle;
    USDC usdc;
    ETH eth;

    function setUp() public {
        oracle = new DreamOracle();
        usdc = new USDC();
        eth = new ETH();
        bank = new MyLend(address(usdc), address(eth),address(oracle));
       // vm.deal(address(bank), 10 ether); 
        vm.deal(address(this), 100 ether);
        

    }

    function testDepositBasic1() public {
        address depositor = address(0x11);
        address borrower = address(0x12);
        
        eth.mint(address(this), 100 ether);
        eth.approve(address(bank), 100 ether);
        bank.deposit(address(eth), 100 ether);
        usdc.mint(address(bank), 10000000000000 ether);
        oracle.setPrice(address(eth), 1 ether);
        for(uint i=0; i<1000; i++)
        {
            bank.borrow(address(usdc), 49 ether);
        }
        usdc.balanceOf(address(this));
    }
} 
```

## rkdnjs / lending / 무한 대출
### 설명
Lending.sol / borrow
이 경우도 위와 마찬가지로 borrow 함수는 대출자의 담보를 확인하고 담보의 50% 미만의 대출금을 빌려주는데, 대출을 하고나서 따로 이 사람이 대출을 했는지에 대한 검사를 하지 않아 한번 넣은 담보로 계속 대출을 할 수 있는 문제점이 발견되었다. 거래소의 존재하는 모든 usdc를 빼올 수 있다.
### 파급력
dex의 모든 usdc를 빼낼 수 있기 때문에 Critical이다.
### 해결 방안
대출을 해주었을 때 담보금에 대한 체크를 하는 체크 루틴을 따로 추가하여 담보금이 재사용되는 것을 방지해야 한다. aave 같은 경우 이를 막기 위해서 debt 토큰을 따로 만들고 관리하여 debt 토큰의 발행량으로 이를 체크한다.
``` pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import "../src/ERC20.sol";
import "../src/Lending.sol";
import "../src/DreamOracle.sol";

contract MintableToken is ERC20 {
    constructor(string memory name, string memory symbol) ERC20(name, symbol) {

    }

    function mint(address receiver, uint256 value) public {
        super._mint(receiver, value);
    }
}

contract lending is Test {
    Lending lending;
    MintableToken ETHER;
    MintableToken USDOLLAR;
    DreamOracle oracle;

    function setUp() public {
        ETHER = new MintableToken("ETHER", "ETH");
        USDOLLAR = new MintableToken("USDOLLAR", "USDC");
        oracle = new DreamOracle();
        oracle.setPrice(address(USDOLLAR), 1000 ether);
        oracle.setPrice(address(ETHER), 1 ether);
        ETHER.mint(address(this), 500 ether);
        USDOLLAR.mint(address(this), 500 ether);
        USDOLLAR.mint(address(lending), 10000000000000000000000000 ether);
        lending = new Lending(address(ETHER), address(USDOLLAR), address(oracle));            
    }

    function testSimpleDeposit() public {
        lending.deposit(address(ETHER), 100 ether);
        for(uint i=0; i<1000; i++)
        {
            lending.borrow(address(ETHER), 1);
        }
        USDOLLAR.balanceOf(address(this));
    }
}```
