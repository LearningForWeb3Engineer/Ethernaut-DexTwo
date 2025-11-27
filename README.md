# Ethernaut-DexTwo
Ethernaut DexTwo解題思路

    // SPDX-License-Identifier: MIT
    pragma solidity ^0.8.0;

    import "openzeppelin-contracts-08/token/ERC20/IERC20.sol";
    import "openzeppelin-contracts-08/token/ERC20/ERC20.sol";
    import "openzeppelin-contracts-08/access/Ownable.sol";

    contract DexTwo is Ownable {
        address public token1;
        address public token2;

    constructor() {}

    function setTokens(address _token1, address _token2) public onlyOwner {
        token1 = _token1;
        token2 = _token2;
    }

    function add_liquidity(address token_address, uint256 amount) public onlyOwner {
        IERC20(token_address).transferFrom(msg.sender, address(this), amount);
    }

    function swap(address from, address to, uint256 amount) public {
        require(IERC20(from).balanceOf(msg.sender) >= amount, "Not enough to swap");
        uint256 swapAmount = getSwapAmount(from, to, amount);
        IERC20(from).transferFrom(msg.sender, address(this), amount);
        IERC20(to).approve(address(this), swapAmount);
        IERC20(to).transferFrom(address(this), msg.sender, swapAmount);
    }

    function getSwapAmount(address from, address to, uint256 amount) public view returns (uint256) {
        return ((amount * IERC20(to).balanceOf(address(this))) / IERC20(from).balanceOf(address(this)));
    }

    function approve(address spender, uint256 amount) public {
        SwappableTokenTwo(token1).approve(msg.sender, spender, amount);
        SwappableTokenTwo(token2).approve(msg.sender, spender, amount);
    }

    function balanceOf(address token, address account) public view returns (uint256) {
        return IERC20(token).balanceOf(account);
    }
    }

    contract SwappableTokenTwo is ERC20 {
        address private _dex;

    constructor(address dexInstance, string memory name, string memory symbol, uint256 initialSupply)
        ERC20(name, symbol)
    {
        _mint(msg.sender, initialSupply);
        _dex = dexInstance;
    }

    function approve(address owner, address spender, uint256 amount) public {
        require(owner != _dex, "InvalidApprover");
        super._approve(owner, spender, amount);
    }
    }

***這題主要是利用swap函式可以接收自己定義的token當作攻擊重點***

因swap(from, to, amount) 沒有檢查 from 是否為 DEX 預期的 token（例如 token1 或 token2）。

getSwapAmount(from,to,amount) = amount * balance(to,DEX) / balance(from,DEX) 使用 DEX 內的實際餘額計算比率。

如果 balance(from,DEX) 非常小，則 swapAmount 會非常大 —— 攻擊者只要把 from（fake token）在 DEX 的餘額做小量的注入，就能用極小的 amount 換走大量 to token。

Tips:解決辦法

1.使用固定價格公式或不可受外部任意 token 影響的定價：例如使用恒定乘積（x*y=k）或其他不可被任意 token 供給扭曲的價格策略。

2.白名單 / 僅允許指定 token：在 swap 中強制 require(from == token1 || from == token2) 並同理檢查 to

3.不要在同一合約中用 approve(this) 並立即 transferFrom 的怪異流程：追求最小權限原則與明確授權流程
