# DEX_Audit

---

# 1. Entropy

• [https://github.com/Entropy1110/Dex_solidity](https://github.com/Entropy1110/Dex_solidity)

commit v : [fb4723c](https://github.com/Entropy1110/Dex_solidity/commit/fb4723cf0af9663378466e639eb7f8eabab19e7d)  

### 1. 유동성 제거

### 설명

Dex.sol/swap

```solidity
        if (amountX > 0) {
            newReserveX = reserveX + amountX;
            newReserveY = (reserveX * reserveY) / newReserveX;
            amountOut = reserveY - newReserveY;

            amountOut = (amountOut * 999) / 1000; // 0.1% fee
            require(amountOut >= minAmountOut, "Insufficient output amount");
            
            tokenX.transferFrom(msg.sender, address(this), amountX);
            tokenY.transfer(msg.sender, amountOut);
        } 
```

`swap` 함수에서, 만약 amountX를 매우 큰 숫자로 넘겨준다면, newReserveX가 매우 큰 값으로 변한다. 이 때, newReserveY는 0이 될 가능성이 있고, amountOut에서 reserveY - newReserveY는 reserveY, 즉 유동성 풀에 예치된 모든 토큰 y에 대해서 뺄 수 있어 유동성 풀을 망가뜨릴 수 있다. 

### 파급력

Level : `Critical`

 한쪽 유동성을 완전히 제거할 수 있다.

### 해결방안

CPMM 비율이 깨지거나, 한쪽이 0이 되는 것에 대한 조건을 걸어, 한번에 큰 자금이 몰려와도 revert로 대응할 수 있게 한다.

---

# 2. Kenny

• [https://github.com/55hnnn/DEX_solidity](https://github.com/55hnnn/DEX_solidity)

commit v : [67b860e](https://github.com/55hnnn/DEX_solidity/commit/67b860e5c79668265596fe5997510bd0dcb4970e)

### 1. LP Token 지급 문제

### 설명

Dex.sol/addLiquidity, line 24.

```solidity
        if (totalLiquidity == 0) {
            // 초기 유동성 공급
            liquidityMinted = amountX * amountY;
        } else {
            // 기존 유동성 공급 시, 현재 비율에 맞춰 유동성 토큰 계산
            uint256 liquidityX = (amountX * totalLiquidity) / tokenX.balanceOf(address(this));
            uint256 liquidityY = (amountY * totalLiquidity) / tokenY.balanceOf(address(this));
            liquidityMinted = liquidityX < liquidityY ? liquidityX : liquidityY;
        }
```

처음 유동성을 제공할 때, liquidityMinted를 단순히 x * y로 하면 너무 숫자가 커져 문제가 될 수 있다.

### 파급력

Level : `info`

부정확한 LP Token 발행으로 문제 가능성 있음

### 해결방안

x * y에 sqrt 함수를 적용한 것을 초기 LP Token 발행량으로 잡아야 함.

### 2. 유동성 제거

### 설명

Dex.sol/swap line 80

```solidity
        if (amountX > 0) {
            // x -> y
            uint256 newTokenXBalance = tokenXBalance + amountX;
            uint256 newTokenYBalance = (tokenXBalance * tokenYBalance) / newTokenXBalance;

            output = (tokenYBalance - newTokenYBalance) * 999 / 1000;

            require(output >= minOutput, "Insufficient output amount");
            require(tokenX.transferFrom(msg.sender, address(this), amountX), "Token X transfer failed");
            require(tokenY.transfer(msg.sender, output), "Token Y transfer failed");
        } 
```

`swap` 함수에서, 만약 amountX를 매우 큰 숫자로 넘겨준다면, newTokenXBalance가 매우 큰 값으로 변한다. 이 때, newTokenYBalance는 0이 될 가능성이 있고, output에서 tokenYBalance - newTokenYBalance로 인해 한 쪽 유동성 풀이 사라져 문제가 발생할 가능성이 있다.

### 파급력

Level : `Critical`

 한쪽 유동성을 완전히 제거할 수 있다.

### 해결방안

CPMM 비율이 깨지거나, 한쪽이 0이 되는 것에 대한 조건을 걸어, 한번에 큰 자금이 몰려와도 revert로 대응할 수 있게 한다.

---

# 3. jacqueline

• [https://github.com/je1att0/DEX_solidity](https://github.com/je1att0/DEX_solidity)

commit v : [a5981c8](https://github.com/je1att0/DEX_solidity/commit/a5981c8297cdee9ab8aae232a32743319445efe5)

### 1. LP 무한 사용

### 설명

Dex.sol/removeLiquidity, line 49.

```solidity
    function removeLiquidity(uint _lpReturn, uint _minAmountX, uint _minAmountY) public returns (uint _tx, uint _ty) {
        uint poolAmountX = tokenX.balanceOf(address(this));
        uint poolAmountY = tokenY.balanceOf(address(this));
        _tx = poolAmountX*_lpReturn/totalLP;
        _ty = poolAmountY*_lpReturn/totalLP;
        require(_tx >= _minAmountX && _ty >= _minAmountY, "Insufficient minimum amount");
        require(_lpReturn <= poolAmountX + poolAmountY, "Insufficient LP return");

        tokenX.transfer(msg.sender, _tx);
        tokenY.transfer(msg.sender, _ty);
        totalLP -= _lpReturn;

        return (_tx, _ty);

    }
```

msg.sender의 LP Token에 대해 removeLiquidity에서 소각해야 하나 그러한 과정이 없어 사실상 지급 받은 LP Token을 무한으로 사용할 수 있음.

### 파급력

Level: `Critical`

Lp token을 무한으로 사용 가능하다.

### 해결방안

LP Token을 removeLiquidity에서 소각해야 함.

### 2. 개발 실수

Dex.sol/swapXtoY, line 74.

```solidity
    function swapXtoY(uint _amountX,  uint _amountY, uint _minOutput) public returns (uint swapReturn) {
        uint poolAmountX = tokenX.balanceOf(address(this));
        uint poolAmountY = tokenY.balanceOf(address(this));
        swapReturn = poolAmountY - poolAmountX*poolAmountY/(poolAmountX + _amountX);
        swapReturn = swapReturn * 999 / 1000;
        require(swapReturn >= _minOutput, "Insufficient minimum output");
        tokenX.transferFrom(msg.sender, address(this), _amountX);
        tokenY.transfer(msg.sender, swapReturn);
        return swapReturn;
    }
```

### 설명

swapXtoY가 swap 내부 인터널 call이므로 public일 이유가 없다. 또한 _amountY의 경우 해당 함수에서 사용하지 않는 파라미터이므로, 딱히 기술할 필요가 없다. 반대인 swapYtoX도 마찬가지.

### 파급력

Level: `info`

불필요한 가스비 지출.

### 해결방안

불필요한 파라미터 없애고, internal로 접근 제어 제한.

### 3. 유동성 제거

### 설명

Dex.sol/swapXtoY line 74

```solidity
    function swapXtoY(uint _amountX,  uint _amountY, uint _minOutput) public returns (uint swapReturn) {
        uint poolAmountX = tokenX.balanceOf(address(this));
        uint poolAmountY = tokenY.balanceOf(address(this));
        swapReturn = poolAmountY - poolAmountX*poolAmountY/(poolAmountX + _amountX);
        swapReturn = swapReturn * 999 / 1000;
        require(swapReturn >= _minOutput, "Insufficient minimum output");
        tokenX.transferFrom(msg.sender, address(this), _amountX);
        tokenY.transfer(msg.sender, swapReturn);
        return swapReturn;
    }
```

`swap` 함수에서, 한쪽 풀에 큰 자금을 넣으면, 반대 유동성을 제거할 수 있다.

### 파급력

Level : `Critical`

 한쪽 유동성을 완전히 제거할 수 있다.

### 해결방안

CPMM 비율이 깨지거나, 한쪽이 0이 되는 것에 대한 조건을 걸어, 한번에 큰 자금이 몰려와도 revert로 대응할 수 있게 한다.

---

# 4. Mia

• [https://github.com/ooMia/Upside_DEX_solidity](https://github.com/ooMia/Upside_DEX_solidity)

commit v : [3dc4e18](https://github.com/ooMia/Upside_DEX_solidity/commit/3dc4e1869aa0e942963de705450de1ee9d25f70a)

### 1.  LP Token 비율 문제

### 설명

Dex.sol/addLiquidity line 70

```solidity
lpAmount = Math.max(uint256(int256(amountX) + dy), uint256(int256(amountY) + dx));
```

처음부터 LP Token 비율을 둘 중 더 큰쪽으로 준다면, 기하 평균에서 멀어질 수 있다.

### 파급력

Level: `info`

LP Token 비율이 유동성 풀 비율과 상이해질 수 있다.

### 해결방안

sqrt(x * y)

### 2. LP Token 단순 변수로 처리

### 설명

Lp Token을 ERC20이 아닌 단순 lpBalances 변수로만 처리하여, LP Token의 활용성을 떨어뜨린다.

### 파급력

Level: `info`

LP Token을 토큰으로 관리하지 않아 활용방안이 없다.

### 해결방안

LP Token을 ERC20 토큰으로 지급한다.

### 3. 유동성 제거

### 설명

Dex.sol/swap line 109

```solidity
    function swap(uint256 amountX, uint256 amountY, uint256 minReturn) external override returns (uint256 amount) {
        if (amountY == 0 && amountX > 0) {
            amount = swapX(amountX);
        } else if (amountX == 0 && amountY > 0) {
            amount = swapY(amountY);
        } else {
            revert("Invalid swap amount");
        }
        require(amount >= minReturn, "Invalid swap amount");
    }
```

`swap` 함수에서, 한쪽 풀에 큰 자금을 넣으면, 반대 유동성을 제거할 수 있다.

### 파급력

Level : `Critical`

 한쪽 유동성을 완전히 제거할 수 있다.

### 해결방안

CPMM 비율이 깨지거나, 한쪽이 0이 되는 것에 대한 조건을 걸어, 한번에 큰 자금이 몰려와도 revert로 대응할 수 있게 한다.

---

# 5. hakid29

• [https://github.com/hakid29/Dex_solidity](https://github.com/hakid29/Dex_solidity)

commit v : [0f49cc9](https://github.com/hakid29/Dex_solidity/commit/0f49cc9c5179fd611c90028dc67f8a10378d23b8)

### 1. FEE_RATE Storage 사용으로 인한 가스비 낭비

### 설명

Dex.sol line 14

```solidity
uint public constant FEE_RATE = 999; // 0.1% fee (1000 - 999)
```

FEE_RATE를 스토리지 변수로 사용해 불필요한 가스비가 발생한다. 추후 수수료 업그레이드를 원한다면 setter 함수를 구현해 놓아야 하거나, 고정으로 간다면 굳이 변수로 선언할 필요가 없다.

### 파급력

Level : `info`

불필요하게 가스비가 낭비될 수 있다.

### 해결방안

수수료를 스토리지 변수로 사용하지 않는다.

### 2. 유동성 제거

### 설명

Dex.sol/swap line 76

`swap` 함수에서, 한쪽 풀에 큰 자금을 넣으면, 반대 유동성을 제거할 수 있다.

### 파급력

Level : `Critical`

 한쪽 유동성을 완전히 제거할 수 있다.

### 해결방안

CPMM 비율이 깨지거나, 한쪽이 0이 되는 것에 대한 조건을 걸어, 한번에 큰 자금이 몰려와도 revert로 대응할 수 있게 한다.

---

# 6. Icarus

• [https://github.com/pluto1011/-Lending-DEX-_solidity](https://github.com/pluto1011/-Lending-DEX-_solidity)

commit v : [df61bf1](https://github.com/pluto1011/-Lending-DEX-_solidity/commit/df61bf1d26cf2d0ec6059c2cacc83e6bbffb45e5)

### 1. 불필요한 변수 선언

### 설명

Dex.sol line 15

```solidity
    uint256 private constant MINIMUM_LIQUIDITY = 1000;
```

사용하지 않는 MINIMUM_LIQUIDITY 선언.

### 파급력

Level : `info`

사용하지 않는 변수 사용.

### 해결방안

해당 변수를 유의미하게 사용하거나 삭제.

### 2. 유동성 제거

### 설명

Dex.sol/swap line 84

`swap` 함수에서, 한쪽 풀에 큰 자금을 넣으면, 반대 유동성을 제거할 수 있다.

### 파급력

Level : `Critical`

 한쪽 유동성을 완전히 제거할 수 있다.

### 해결방안

CPMM 비율이 깨지거나, 한쪽이 0이 되는 것에 대한 조건을 걸어, 한번에 큰 자금이 몰려와도 revert로 대응할 수 있게 한다.

### 3. LP Token 단순 변수로 처리

### 설명

Lp Token을 ERC20으로 사용하지 않아 토큰 활용성을 낮춤

### 파급력

Level: `info`

LP Token을 토큰으로 관리하지 않아 활용방안이 없다.

### 해결방안

LP Token을 ERC20 토큰으로 지급한다. 혹은 DEX 자체를 ERC20을 상속받아 구현한다.

---

# 7. Damon

- [https://github.com/gloomydumber/DEX_solidity](https://github.com/gloomydumber/DEX_solidity)

commit v : [1671f2f](https://github.com/gloomydumber/DEX_solidity/commit/1671f2f3a6a272387b4a884e92f9724563d9cb15)

### 1. Improper Uint Type Casting

### 설명

Dex.sol/update line 138

```solidity
    function update(uint256 _balanceX, uint256 _balanceY) public {
        require(_balanceX <= type(uint112).max && _balanceY <= type(uint112).max, "OVERFLOW");
        reserveX = uint112(_balanceX);
        reserveY = uint112(_balanceY);
    }
```

addLiquidity에서 uint256으로 아무런 제한 없이 유동성을 받지만, update에서 reserve를 업데이트할 때, uint112로 타입캐스팅하여 의도치 않게 많은 토큰을 보낸 사람이 손해를 볼 수 있다.

### 파급력

Level: `Medium`

실용적이진 않으나 매우 큰 토큰을 보내는 사람은 엄청난 손해를 볼 수 있다.

### 해결방안

유동성을 제공할때부터 uint112에 대한 체크를 추가한다.

### 2. update에 대한 부적절한 접근 제어로 유동성 제공 불가

### 설명

Dex.sol line 138.

```solidity
    function update(uint256 _balanceX, uint256 _balanceY) public {
        require(_balanceX <= type(uint112).max && _balanceY <= type(uint112).max, "OVERFLOW");
        reserveX = uint112(_balanceX);
        reserveY = uint112(_balanceY);
    }
```

update를 누구나 호출할 수 있어 lp token을 받지 못하는 등 잠재적인 문제가 발생할 수 있다. 하나의 예로, update로 악의적으로 reserve 값을 아무 대가 없이 엄청난 수로 늘릴 수 있고, 이렇게 되면 addLiquidity에서 mint함수에서 uint underflow가 발생하여 유동성 제공을 불가능하게 만들 수 있다.

### 파급력

Level: `High`

악의적은 공격자가 reserve 값을 크게 만들어 유동성 제공을 못하게 한다.

### 해결방안

internal call인 update를 internal로 접근제어를 수정한다.

### 3. mint, update에 대한 부적절한 접근제어로 인한 악의적인 mint

### 설명

Dex.sol line 94

```solidity
    function mint(address _to) external returns (uint256 liquidity) {
        (uint112 _reserveX, uint112 _reserveY) = getReserves();
        uint256 balanceX = IERC20(tokenX).balanceOf(address(this));
        uint256 balanceY = IERC20(tokenY).balanceOf(address(this));
        uint256 amountX = balanceX - _reserveX;
        uint256 amountY = balanceY - _reserveY;
        uint256 totalSupply = totalSupply();

        if (totalSupply == 0) {
            liquidity = Math.sqrt(amountX * amountY);
        } else {
            liquidity = Math.min(amountX * totalSupply / reserveX, amountY * totalSupply / reserveY);
        }

        require(liquidity > 0, "INSUFFICIENT_LIQUIDITY_MINTED");

        _mint(_to, liquidity);
        update(balanceX, balanceY);
    }
```

mint와 update가 둘 다 public이다. 공격자는 update로 reserver를 1과 같이 낮은 값으로 설정하고, mint를 다이렉트로 호출하면 엄청난 양의 LP Token을 지급 받을 수 있다.

### 파급력

Level : `Critical`

유동성을 제공하지 않은 사용자가 엄청난 양의 LP Token을 지급 받을 수 있다.

### 해결방안

적절한 접근 제어를 사용한다.

### 4. 유동성 제거

### 설명

Dex.sol/swap line 19

`swap` 함수에서, 한쪽 풀에 큰 자금을 넣으면, 반대 유동성을 제거할 수 있다.

### 파급력

Level : `Critical`

 한쪽 유동성을 완전히 제거할 수 있다.

### 해결방안

CPMM 비율이 깨지거나, 한쪽이 0이 되는 것에 대한 조건을 걸어, 한번에 큰 자금이 몰려와도 revert로 대응할 수 있게 한다.

---

# 8. Muang

• [https://github.com/GODMuang/DEX_solidity](https://github.com/GODMuang/DEX_solidity)

commit v : [d99cfb5](https://github.com/GODMuang/DEX_solidity/commit/d99cfb5e97a8e3b70c1a984fb875d681677e8a51)

### 1. burn 함수에 대한 잘못된 접근제어 및 잘못된 burn으로 인한 자금 탈취

### 설명

Dex.sol line 51

```solidity
    function burn(address to) external lock returns (uint256 amountX, uint256 amountY) {
        uint256 balanceX = IERC20(tokenX).balanceOf(address(this));
        uint256 balanceY = IERC20(tokenY).balanceOf(address(this));
        uint256 totalSupply = totalSupply();
        uint256 liquidity = balanceOf(address(this));

        amountX = liquidity * balanceX / totalSupply;
        amountY = liquidity * balanceY / totalSupply;

        require(amountX > 0 && amountY > 0, "INSUFFICENT_LIQUIDITY_BURNED");
        _burn(address(this), liquidity);
        safeTransfer(address(tokenX), msg.sender, amountX);
        safeTransfer(address(tokenY), msg.sender, amountY);

        balanceX = IERC20(tokenX).balanceOf(address(this));
        balanceY = IERC20(tokenY).balanceOf(address(this));

        update(balanceX, balanceY);

    }
```

burn 함수가 external이라 누구나 호출할 수 있고 또 _burn함수가 자기 자신의 lp token에 대해서만 _burn 하므로 유동성을 제공하지 않은 사용자가 safeTransfer로 자금을 탈취할 수 있다.

### 파급력

Level : `Critical`

공격자가 자유롭게 유동성 풀에 예치된 자금 탈취 가능

### 해결방안

burn에 대한 접근제어를 internal로 하고, _burn의 첫번째 파라미터를 msg.sender로 고쳐야 함.

### 2. 유동성 제거

### 설명

Dex.sol/swap line 84

`swap` 함수에서, 한쪽 풀에 큰 자금을 넣으면, 반대 유동성을 제거할 수 있다.

### 파급력

Level : `Critical`

 한쪽 유동성을 완전히 제거할 수 있다.

### 해결방안

CPMM 비율이 깨지거나, 한쪽이 0이 되는 것에 대한 조건을 걸어, 한번에 큰 자금이 몰려와도 revert로 대응할 수 있게 한다.

---

# 9. bob

• [https://github.com/choihs0457/DEX_solidity.git](https://github.com/choihs0457/DEX_solidity.git)

commit v : 973f72b

### 1. LP Token 활용 불가

### 설명

LP Token을 ERC20이 아닌 단순 내부 변수로 관리하여, 토큰 자체의 활용도가 낮음

### 파급력

Level : `info`

LP Token 활용 불가.

### 해결방안

LP Token을 ERC20으로 발행.

### 2. 유동성 제거

### 설명

Dex.sol/swap line 74

`swap` 함수에서, 한쪽 풀에 큰 자금을 넣으면, 반대 유동성을 제거할 수 있다.

### 파급력

Level : `Critical`

 한쪽 유동성을 완전히 제거할 수 있다.

### 해결방안

CPMM 비율이 깨지거나, 한쪽이 0이 되는 것에 대한 조건을 걸어, 한번에 큰 자금이 몰려와도 revert로 대응할 수 있게 한다.

---

# 10. ella

• [https://github.com/skskgus/Dex_solidity](https://github.com/skskgus/Dex_solidity)

commit v : [1b37751](https://github.com/kaymin128/Dex_solidity/commit/1b37751893d23ec3119147f4e8e6d804fb40d1d1)

### 1. update 남용

Dex.sol/removeLiauidity line 90

```solidity
    function removeLiquidity(uint256 shares, uint256 minAmount0, uint256 minAmount1) external returns (uint256 amount0, uint256 amount1) {
        uint256 _reserve0 = reserve0;
        uint256 _reserve1 = reserve1;
        uint256 _totalSupply = totalSupply;

        amount0 = (shares * _reserve0) / _totalSupply;
        amount1 = (shares * _reserve1) / _totalSupply;

        require(amount0 >= minAmount0, "Insufficient amount0");
        require(amount1 >= minAmount1, "Insufficient amount1");

        _burn(msg.sender, shares);

        _update();

        token0.transfer(msg.sender, amount0);
        token1.transfer(msg.sender, amount1);

        _update();
    }
```

의미 없는 타이밍에 update를 호출해 가스를 낭비

### 파급력

Level : `info`

불필요한 가스 낭비.

### 해결방안

update를 필요할때만 호출하도록 함.

### 2. LP Token 활용 불가

### 설명

LP Token을 ERC20이 아닌 단순 내부 변수로 관리하여, 토큰 자체의 활용도가 낮음

### 파급력

Level : `info`

LP Token 활용 불가.

### 해결방안

LP Token을 ERC20으로 발행.

### 3. 유동성 제거

### 설명

Dex.sol/swap line 111

`swap` 함수에서, 한쪽 풀에 큰 자금을 넣으면, 반대 유동성을 제거할 수 있다.

### 파급력

Level : `Critical`

 한쪽 유동성을 완전히 제거할 수 있다.

### 해결방안

CPMM 비율이 깨지거나, 한쪽이 0이 되는 것에 대한 조건을 걸어, 한번에 큰 자금이 몰려와도 revert로 대응할 수 있게 한다.

---