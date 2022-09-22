.. index:: contract;modular, modular contract

*********************
库合约使用
*********************

通过在合约中引入模块化方法，可以帮助我们减少复杂度以及提高可读性，可以帮助我们定位 bug，发现开发和代码评审期间的漏洞。

在下面的例子中，合约使用 ``Balances`` 的 ``move`` 方法 :ref:`library <libraries>`来检查地址之间发送的余额是否符合你的期望。
通过这种方式， ``Balances`` 库提供了一个孤立的组件，正确地跟踪账户的余额。
很容易验证 ``Balances`` 库永远不会产生负的余额或溢出，所有余额的总和在合约的有效期内是一个不变量。


.. code-block:: solidity

    // SPDX-License-Identifier: GPL-3.0
    pragma solidity >=0.5.0  <0.9.0;

    library Balances {
        function move(mapping(address => uint256) storage balances, address from, address to, uint amount) internal {
            require(balances[from] >= amount);
            require(balances[to] + amount >= balances[to]);
            balances[from] -= amount;
            balances[to] += amount;
        }
    }

    contract Token {
        mapping(address => uint256) balances;
        using Balances for *;   // 引入库
        mapping(address => mapping (address => uint256)) allowed;

        event Transfer(address from, address to, uint amount);
        event Approval(address owner, address spender, uint amount);


        function transfer(address to, uint amount) external returns (bool success) {
            balances.move(msg.sender, to, amount);
            emit Transfer(msg.sender, to, amount);
            return true;

        }

        function transferFrom(address from, address to, uint amount) external returns (bool success) {
            require(allowed[from][msg.sender] >= amount);
            allowed[from][msg.sender] -= amount;
            balances.move(from, to, amount);   // 使用了库方法
            emit Transfer(from, to, amount);
            return true;
        }

        function approve(address spender, uint tokens) external returns (bool success) {
            require(allowed[msg.sender][spender] == 0, "");
            allowed[msg.sender][spender] = tokens;
            emit Approval(msg.sender, spender, tokens);
            return true;
        }

        function balanceOf(address tokenOwner) external view returns (uint balance) {
            return balances[tokenOwner];
        }
    }