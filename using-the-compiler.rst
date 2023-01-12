******************
使用编译器
******************

.. index:: ! commandline compiler, compiler;commandline, ! solc

.. _commandline-compiler:

使用命令行编译器
******************************

.. note:: 
  这一节不适用于 :ref:`solcjs <solcjs>` , 即使它是在命令行模式下使用的也不行。


基础用法
-----------

``solc`` 是 Solidity 源码库的构建目标之一，它是 Solidity 的命令行编译器。你可使用 ``solc --help`` 命令来查看它的所有选项的解释。该编译器可以生成各种输出，范围从简单的二进制文件、汇编文件到用于估计“gas”使用情况的抽象语法树（解析树）。如果你只想编译一个文件，你可以运行 ``solc --bin sourceFile.sol`` 来生成二进制文件。
如果你想通过 ``solc`` 获得一些更高级的输出信息，可以通过 ``solc -o outputDirectory --bin --ast-compact-json --asm sourceFile.sol`` 命令将所有的输出都保存到一个单独的文件夹中。


优化器选项
-----------------

在部署合约之前，在编译时使用 ``solc --optimize --bin sourceFile.sol`` 激活优化器。
默认情况下，优化器将使用 200 runs，假设合约在会被调用200次。
如果你想让最初的合约部署更便宜，而后来的函数执行更昂贵，请设置为 ``--optimize-runs=1``。
如果你期望合约有很多次交易，并且不在乎更高的部署成本和输出字节码大小，那么把 ``--optimize-runs`` 设置成一个高的数字。
这个参数对以下方面有影响（将来可能会改变）：

- 函数调度程序（路由匹配函数）中二进制搜索的大小
- 像大数字或字符串等常量的存储方式


.. index:: allowed paths, --allow-paths, base path, --base-path, include paths, --include-path

基本路径和导入重映射
------------------------------------------

命令行编译器会自动从文件系统中读取并导入的文件，但同时，它也支持通过 :ref:`path redirects <import-remapping>` 使用 ``prefix=path`` 选项将路径重定向。比如：

.. code-block:: bash

    solc github.com/ethereum/dapp-bin/=/usr/local/lib/dapp-bin/ file.sol

这实质上是指示编译器去搜索 ``/usr/local/lib/dapp-bin`` 目录下的所有以 ``github.com/ethereum/dapp-bin/`` 开头的文件。


当访问文件系统搜索导入文件时，:ref:`不以./或../开头的路径 <direct-imports>` 被视为相对于使用 ``--base-path`` 和 ``--include-path`` 选项指定的目录（如果没有指定基本路径，则是当前工作目录）。
此外，通过这些选项添加的路径部分将不会出现在合约元数据中。

出于安全考虑，编译器 :ref:`对它可以访问的目录有一些限制 <allowed-paths>`。
在命令行中指定的源文件的目录和重映射的目标路径被自动允许文件阅读器访问，但其他的都是默认为拒绝的。

通过 ``--allow-paths /sample/path,/another/sample/path`` 语句可以允许额外的路径（和它们的子目录）。
通过 ``--base-path`` 指定的路径内的所有内容都是允许的。

以上只是对编译器如何处理导入路径的一个简化。
关于详细的解释，包括例子和边缘情况的讨论，请参考 :ref:`路径解析 <path-resolution>` 一节。


.. index:: ! linker, ! --link, ! --libraries

.. _library-linking:

库链接
---------------


如果你的合约使用 :ref:`库 <libraries>`，
你会注意到字节码中含有 ``__$53aea86b7d70b31448b230b20ae141a537$__`` 形式的字符串。
这些是实际库的地址的占位符。此占位符是完全限定库名的keccak256 哈希的十六进制编码的34个字符前缀。
字节码文件也将包含形式为 ``// <placeholder> -> <fq library name>`` 的代码行，以帮助识别占位符代表的库。
注意，完全限定的库名是其源文件的路径和用 ``:`` 分隔的库名称。
你可以使用 ``solc`` 作为链接器，意味着你将在这些地方插入库的地址：


要么在你的命令中加入 ``--libraries "file.sol:Math=0x1234567890123456789012345678901234567890 file.sol:Heap=0xabCD567890123456789012345678901234567890"`` 为每个库提供一个地址（用逗号或空格作为分隔符），
要么用 ``-libraries fileName`` 运行 ``solc`` 将字符串存储在一个文件中（每行一个库）。


.. note::
    从Solidity 0.8.1 开始，接受 ``=`` 作为库和地址之间的分隔符，而 ``:`` 作为分隔符已被废弃。
    它将在未来被删除。目前 ``--libraries "file.sol:Math:0x1234567890123456789012345678901234567890 file.sol:Heap:0xabCD56789012345678901234567890"`` 也可以工作。

.. index:: --standard-json, --base-path

如果调用 ``solc`` 时使用 ``--standard-json`` 选项，它将在标准输入中期待一个JSON 输入（下面解释），
并在标准输出中返回一个JSON输出。在更复杂的、及自动化使用时的推荐接口。
该进程将始终以 “成功（success）” 状态终止，并通过JSON输出来报告任何错误。
选项 ``--base-path`` 也以标准JSON模式处理。

如果调用 ``solc`` 时使用 ``--link`` 选项，所有输入文件都被编译成格式为 ``__$53aea86b7d70b31448b230b20ae141a537$__`` 形式的未链接的二进制文件（十六进制编码），
并就地链接（如果从标准输入(stdin)读取输入，则被写到标准输出(stdout)）。
在这种情况下，除了 ``--libraries`` 以外的所有选项都被忽略（包括 ``-o`` ）。

.. warning::
    
    不推荐在生成的字节码上手动链接库文件，因为它不会更新合约元数据。
    由于元数据包含在编译时指定的库的列表，而字节码包含元数据哈希，
    你将得到不同的二进制文件，并且这取决于何时进行链接。
    
    你应该在编译合约时请求编译器链接库文件，方法是使用 ``solc`` 的 ``--libraries`` 选项
    或 ``libraries`` 键（如果你使用编译器的标准JSON接口）。

.. note::
    库的占位符曾经是库本身的完全限定名称，而不是它的哈希值。
    这种格式仍然被 ``solc --link`` 支持，但编译器将不再输出它。
    这一改变是为了减少库之间发生碰撞的可能性，因为只有完全限定的库名的前36个字符可以被使用。


.. _evm-version:
.. index:: ! EVM version, compile target

将EVM版本设置为目标版本
*********************************

当你编译你的合约代码时，你可以指定以太坊虚拟机版本来编译代码，以避免特定的功能或行为。

.. warning::
   在错误的EVM版本进行编译会导致错误、奇怪和失败的行为。
   请确保，特别是在运行一个私有链的情况下，你使用了匹配的EVM版本。

在命令行中，你可以选择EVM的版本，方法如下所示：

.. code-block:: shell

  solc --evm-version <VERSION> contract.sol

在 :ref:`标准 JSON 接口 <compiler-api>` 中，使用 ``"settings"`` 字段中的键 ``"evmVersion"``。


.. code-block:: javascript

    {
      "sources": {/* ... */},
      "settings": {
        "optimizer": {/* ... */},
        "evmVersion": "<VERSION>"
      }
    }

EVM版本选项
--------------

以下是 EVM 版本的列表，以及每个版本中引入的编译器相关变化。
每个版本之间不保证向后兼容。


- ``homestead``
   - （最老的版本）
- ``tangerineWhistle``
   - 访问其他账户的gas成本增加，与gas估算和优化器有关。
   - 对于外部调用，所有gas都是默认发送的，以前必须保留一定的数量。
- ``spuriousDragon``
   -  ``exp`` 操作码的gas成本增加，与gas估计和优化器有关。
- ``byzantium``
   - 在汇编中可使用操作码 ``returndatacopy``， ``returndatasize`` 和 ``staticcall``。
   - ``staticcall`` 操作码在调用非库合约 view 或 pure 函数时使用，它可以防止函数在 EVM 级别修改状态，也就是说，甚至适用于无效的类型转换的情况。
   - 可以访问从函数调用返回的动态数据。
   -  引入了 ``revert`` 操作码，这意味着 ``revert()`` 将不会浪费gas。
- ``constantinople``
   - 在汇编中可使用操作码 ``create2`` ， ``extcodehash`` ， ``shl`` ， ``shr`` 和 ``sar``。
   - 移位运算符使用移位运算码，因此需要的gas较少。
- ``petersburg``
   - 编译器的行为与 constantinople 版本的行为相同。
- ``istanbul``
   - 在汇编中可使用操作码 ``chainid`` 和 ``selfbalance``。
- ``berlin``
   - ``SLOAD`` ， ``*CALL`` ， ``BALANCE`` ， ``EXT*`` 和 ``SELFDESTRUCT`` 的gas成本增加。
     编译器假设这类操作的gas成本是固定的。这与gas估计和优化器有关。
- ``london`` (**默认**)
   -  可以通过全局的 ``block.basefee`` 或内联汇编中的 ``basefee()`` 访问区块的基本费用（base fee），参考： (`EIP-3198 <https://eips.ethereum.org/EIPS/eip-3198>`_ 和 `EIP-1559 <https://eips.ethereum.org/EIPS/eip-1559>`_) 


.. index:: ! standard JSON, ! --standard-json

.. _compiler-api:

编译器输入输出JSON描述
******************************************


对于更复杂和自动化的设置，可以使用JSON输入输出接口，编译器的所有发行版都提供相同的接口。

这些字段可能会发生变化，有些字段是可选的（如前所述），但我们尽量只做向后兼容的改动。

编译器API期望JSON格式的输入，并将编译结果输出为JSON格式的输出。
不使用标准错误输出，进程将始终以 “成功” 状态终止，即使存在错误。错误总是作为JSON输出的一部分报告。


以下各小节通过一个例子来描述该格式。
注释是不允许的，这里仅用于解释目的。

输入说明
-----------------

.. code-block:: javascript

    {
      // 必选: 源代码语言，比如“Solidity”，“serpent”，“lll”，“assembly”等
      language: "Solidity",
      // 必选
      sources:
      {
        // 这里的键值是源文件的“全局”名称，可以通过remappings引入其他文件（参考下文）
        "myFile.sol":
        {
          // 可选: 源文件的kaccak256哈希值，可用于校验通过URL加载的内容。
          "keccak256": "0x123...",
          // 必选（除非声明了 "content" 字段）: 指向源文件的URL。
          // URL(s) 会按顺序加载，并且结果会通过keccak256哈希值进行检查（如果有keccak256的话）
          // 如果哈希值不匹配，或者没有URL返回成功，则抛出一个异常。
          "urls":
          [
            "bzzr://56ab...",
            "ipfs://Qma...",
            "file:///tmp/path/to/file.sol"
          ]
        },
        "mortal":
        {
          // 可选: 该文件的keccak256哈希值
          "keccak256": "0x234...",
          // 必选（除非声明了 "urls" 字段）: 源文件的字面内容
          "content": "contract mortal is owned { function kill() { if (msg.sender == owner) selfdestruct(owner); } }"
        }
      },
      // 可选
      settings:
      {
        // 可选: Stop compilation after the given stage. Currently only "parsing" is valid here
        "stopAfter": "parsing",
        // 可选: 重定向参数的排序列表
        remappings: [ ":g/dir" ],
        // 可选: 优化器配置
        optimizer: {
          // 默认为 disabled
          // NOTE: enabled=false still leaves some optimizations on. See comments below.
          // WARNING: Before version 0.8.6 omitting the 'enabled' key was not equivalent to setting
          // it to false and would actually disable all the optimizations.
          enabled: true,
          // 基于你希望运行多少次代码来进行优化。
          // 较小的值可以使初始部署的费用得到更多优化，较大的值可以使高频率的使用得到优化。
          runs: 200,
          // Switch optimizer components on or off in detail.
          // The "enabled" switch above provides two defaults which can be
          // tweaked here. If "details" is given, "enabled" can be omitted.
          "details": {
            // The peephole optimizer is always on if no details are given,
            // use details to switch it off.
            "peephole": true,
            // The inliner is always on if no details are given,
            // use details to switch it off.
            "inliner": true,
            // The unused jumpdest remover is always on if no details are given,
            // use details to switch it off.
            "jumpdestRemover": true,
            // Sometimes re-orders literals in commutative operations.
            "orderLiterals": false,
            // Removes duplicate code blocks
            "deduplicate": false,
            // Common subexpression elimination, this is the most complicated step but
            // can also provide the largest gain.
            "cse": false,
            // Optimize representation of literal numbers and strings in code.
            "constantOptimizer": false,
            // The new Yul optimizer. Mostly operates on the code of ABI coder v2
            // and inline assembly.
            // It is activated together with the global optimizer setting
            // and can be deactivated here.
            // Before Solidity 0.6.0 it had to be activated through this switch.
            "yul": false,
            // Tuning options for the Yul optimizer.
            "yulDetails": {
              // Improve allocation of stack slots for variables, can free up stack slots early.
              // Activated by default if the Yul optimizer is activated.
              "stackAllocation": true,
              // Select optimization steps to be applied.
              // Optional, the optimizer will use the default sequence if omitted.
              "optimizerSteps": "dhfoDgvulfnTUtnIf..."
            }
        },
        // 指定需编译的EVM的版本。会影响代码的生成和类型检查。可用的版本为：homestead，tangerineWhistle，spuriousDragon，byzantium，constantinople
        evmVersion: "byzantium",
        // Optional: Change compilation pipeline to go through the Yul intermediate representation.
        // This is false by default.
        "viaIR": true,
        // Optional: Debugging settings
        "debug": {
          // How to treat revert (and require) reason strings. Settings are
          // "default", "strip", "debug" and "verboseDebug".
          // "default" does not inject compiler-generated revert strings and keeps user-supplied ones.
          // "strip" removes all revert strings (if possible, i.e. if literals are used) keeping side-effects
          // "debug" injects strings for compiler-generated internal reverts, implemented for ABI encoders V1 and V2 for now.
          // "verboseDebug" even appends further information to user-supplied revert strings (not yet implemented)
          "revertStrings": "default",
          // Optional: How much extra debug information to include in comments in the produced EVM
          // assembly and Yul code. Available components are:
          // - `location`: Annotations of the form `@src <index>:<start>:<end>` indicating the
          //    location of the corresponding element in the original Solidity file, where:
          //     - `<index>` is the file index matching the `@use-src` annotation,
          //     - `<start>` is the index of the first byte at that location,
          //     - `<end>` is the index of the first byte after that location.
          // - `snippet`: A single-line code snippet from the location indicated by `@src`.
          //     The snippet is quoted and follows the corresponding `@src` annotation.
          // - `*`: Wildcard value that can be used to request everything.
          "debugInfo": ["location", "snippet"]
        },
        // 可选: 元数据配置
        metadata: {
          // 只可使用字面内容，不可用URLs （默认设为 false）
          useLiteralContent: true,
          // Use the given hash method for the metadata hash that is appended to the bytecode.
          // The metadata hash can be removed from the bytecode via option "none".
          // The other options are "ipfs" and "bzzr1".
          // If the option is omitted, "ipfs" is used by default.
          "bytecodeHash": "ipfs"
        },
        // 库的地址。如果这里没有把所有需要的库都给出，会导致生成输出数据不同的未链接对象
        libraries: {
          // 最外层的 key 是使用这些库的源文件的名字。
          // 如果使用了重定向， 在重定向之后，这些源文件应该能匹配全局路径
          // 如果源文件的名字为空，则所有的库为全局引用
          "myFile.sol": {
            "MyLib": "0x123123..."
          }
        },
        // 以下内容可以用于选择所需的输出。
        // 如果这个字段被忽略，那么编译器会加载并进行类型检查，但除了错误之外不会产生任何输出。
        // 第一级的key是文件名，第二级是合约名称，如果合约名为空，则针对文件本身（进行输出）。
        // 若使用通配符*，则表示所有合约。
        //
        // 可用的输出类型如下所示：
        //   abi - ABI
        //   ast - 所有源文件的AST
        //   devdoc - 开发者文档（natspec）
        //   userdoc - 用户文档（natspec）
        //   metadata - 元数据
        //   ir - 去除语法糖（desugaring）之前的新汇编格式
        //   irOptimized - Intermediate representation after optimization
        //   storageLayout - Slots, offsets and types of the contract's state variables.
        //   evm.assembly - 去除语法糖（desugaring）之后的新汇编格式
        //   evm.legacyAssembly - JSON的旧样式汇编格式
        //   evm.bytecode.functionDebugData - Debugging information at function level
        //   evm.bytecode.object - 字节码对象
        //   evm.bytecode.opcodes - 操作码列表
        //   evm.bytecode.sourceMap - 源码映射（用于调试）
        //   evm.bytecode.linkReferences - 链接引用（如果是未链接的对象）
        //   evm.bytecode.generatedSources - Sources generated by the compiler
        //   evm.deployedBytecode* - 部署的字节码（具有evm.bytecode所有的选项）
        //   evm.deployedBytecode.immutableReferences - Map from AST ids to bytecode ranges that reference immutables
        //   evm.methodIdentifiers - 函数哈希值列表
        //   evm.gasEstimates - 函数的gas预估量
        //   ewasm.wast - Ewasm in WebAssembly S-expressions 格式
        //   ewasm.wasm - Ewasm in WebAssembly 二进制格式
        //
        // 请注意，如果使用 `evm` ，`evm.bytecode` ，`ewasm` 等选项，会选择其所有的子项作为输出。 另外，`*`可以用作通配符来请求所有内容。
        //
        outputSelection: {
          // 为每个合约生成元数据和字节码输出。
          "*": {
            "*": [ "metadata"，"evm.bytecode" ]
          },
          // 启用“def”文件中定义的“MyContract”合约的abi和opcodes输出。
          "def": {
            "MyContract": [ "abi"，"evm.bytecode.opcodes" ]
          },
      },
      // The modelChecker object is experimental and subject to changes.
        "modelChecker":
        {
          // Chose which contracts should be analyzed as the deployed one.
          "contracts":
          {
            "source1.sol": ["contract1"],
            "source2.sol": ["contract2", "contract3"]
          },
          // Choose how division and modulo operations should be encoded.
          // When using `false` they are replaced by multiplication with slack
          // variables. This is the default.
          // Using `true` here is recommended if you are using the CHC engine
          // and not using Spacer as the Horn solver (using Eldarica, for example).
          // See the Formal Verification section for a more detailed explanation of this option.
          "divModNoSlacks": false,
          // Choose which model checker engine to use: all (default), bmc, chc, none.
          "engine": "chc",
          // Choose which types of invariants should be reported to the user: contract, reentrancy.
          "invariants": ["contract", "reentrancy"],
          // Choose whether to output all unproved targets. The default is `false`.
          "showUnproved": true,
          // Choose which solvers should be used, if available.
          // See the Formal Verification section for the solvers description.
          "solvers": ["cvc4", "smtlib2", "z3"],
          // Choose which targets should be checked: constantCondition,
          // underflow, overflow, divByZero, balance, assert, popEmptyArray, outOfBounds.
          // If the option is not given all targets are checked by default,
          // except underflow/overflow for Solidity >=0.8.7.
          // See the Formal Verification section for the targets description.
          "targets": ["underflow", "overflow", "assert"],
          // Timeout for each SMT query in milliseconds.
          // If this option is not given, the SMTChecker will use a deterministic
          // resource limit by default.
          // A given timeout of 0 means no resource/time restrictions for any query.
          "timeout": 20000
        }
      }
    }


输出说明
------------------

.. code-block:: javascript

    {
      // 可选：如果没有遇到错误/警告/提示，则不出现
      errors: [
        {
          // 可选：源文件中的位置
          sourceLocation: {
            file: "sourceFile.sol",
            start: 0,
            end: 100
          },
        // Optional: Further locations (e.g. places of conflicting declarations)
          secondarySourceLocations: [
            {
              "file": "sourceFile.sol",
              "start": 64,
              "end": 92,
              "message": "Other declaration is here:"
            }
          ],
          // 强制: 错误类型，例如 “TypeError”， “InternalCompilerError”， “Exception”等.
          // 可在文末查看完整的错误类型列表
          type: "TypeError",
          // 强制: 发生错误的组件，例如“general”，“ewasm”等
          component: "general",
          // 强制：错误的严重级别（“error” , “warning” 或 "info"）, 将来也许会扩展
          severity: "error",
          // 可选: 引起错误的唯一编码
          "errorCode": "3141",
          // 强制
          message: "Invalid keyword",
          // 可选: 带错误源位置的格式化消息
          formattedMessage: "sourceFile.sol:100: Invalid keyword"
        }
      ],
      // 这里包含了文件级别的输出。可以通过outputSelection来设置限制/过滤。
      sources: {
        "sourceFile.sol": {
          // 标识符（用于源码映射）
          id: 1,
          // AST对象
          ast: {},
        }
      },
      // 这里包含了合约级别的输出。 可以通过outputSelection来设置限制/过滤。
      contracts: {
        "sourceFile.sol": {
          // 如果使用的语言没有合约名称，则该字段应该留空。
          "ContractName": {
            // 以太坊合约的应用二进制接口（ABI）。如果为空，则表示为空数组。
            // 请参阅 https://docs.soliditylang.org/en/develop/abi-spec.html
            abi: [],
            // 请参阅元数据输出文档（序列化的JSON字符串）
            metadata: "{/* ... */}",
            // 用户文档（natspec）
            userdoc: {},
            // 开发人员文档（natspec）
            devdoc: {},
            // 中间表示形式 (string)
            ir: "",
            // See the Storage Layout documentation.
            "storageLayout": {"storage": [/* ... */], "types": {/* ... */} },
            // EVM相关输出
            evm: {
              // 汇编 (string)
              assembly: "",
              // 旧风格的汇编 (object)
              legacyAssembly: {},
              // 字节码和相关细节
              bytecode: {
                // Debugging data at the level of functions.
                "functionDebugData": {
                  // Now follows a set of functions including compiler-internal and
                  // user-defined function. The set does not have to be complete.
                  "@mint_13": { // Internal name of the function
                    "entryPoint": 128, // Byte offset into the bytecode where the function starts (optional)
                    "id": 13, // AST ID of the function definition or null for compiler-internal functions (optional)
                    "parameterSlots": 2, // Number of EVM stack slots for the function parameters (optional)
                    "returnSlots": 1 // Number of EVM stack slots for the return values (optional)
                  }
                },
                // 十六进制字符串的字节码
                object: "00fe",
                // 操作码列表 (string)
                opcodes: "",
                // 源码映射的字符串。 请参阅源码映射的定义
                sourceMap: "",
                // Array of sources generated by the compiler. Currently only
                // contains a single Yul file.
                "generatedSources": [{
                  // Yul AST
                  "ast": { /* ...*/ },
                  // Source file in its text form (may contain comments)
                  "contents":"{ function abi_decode(start, end) -> data { data := calldataload(start) } }",
                  // Source file ID, used for source references, same "namespace" as the Solidity source files
                  "id": 2,
                  "language": "Yul",
                  "name": "#utility.yul"
                }],
                // 如果这里给出了信息，则表示这是一个未链接的对象
                linkReferences: {
                  "libraryFile.sol": {
                    // 字节码中的字节偏移；链接时，从指定的位置替换20个字节
                    "Library1": [
                      { start: 0，length: 20 },
                      { start: 200，length: 20 }
                    ]
                  }
                }
              },
             
              deployedBytecode: {
                /* ..., */  // 与上面相同的布局
                "immutableReferences": {
                  // There are two references to the immutable with AST ID 3, both 32 bytes long. One is
                  // at bytecode offset 42, the other at bytecode offset 80.
                  "3": [{ "start": 42, "length": 32 }, { "start": 80, "length": 32 }]
                }
              },
              // 函数哈希的列表
              methodIdentifiers: {
                "delegate(address)": "5c19a95c"
              },
              // 函数的gas预估量
              gasEstimates: {
                creation: {
                  codeDepositCost: "420000",
                  executionCost: "infinite",
                  totalCost: "infinite"
                },
                external: {
                  "delegate(address)": "25000"
                },
                internal: {
                  "heavyLifting()": "infinite"
                }
              }
            },
            // Ewasm相关的输出
            ewasm: {
              // S-expressions格式
              wast: "",
              // 二进制格式（十六进制字符串）
              wasm: ""
            }
          }
        }
      }
    }


错误类型
~~~~~~~~~~~

1. ``JSONError``: JSON输入不符合所需格式，例如，输入不是JSON对象，不支持的语言等。
2. ``IOError``: IO和导入处理错误，例如，在提供的源里包含无法解析的URL或哈希值不匹配。
3. ``ParserError``: 源代码不符合语言规则。
4. ``DocstringParsingError``: 注释块中的NatSpec标签无法解析。
5. ``SyntaxError``: 语法错误，例如 ``continue`` 在 ``for`` 循环外部使用。
6. ``DeclarationError``: 无效的，无法解析的或冲突的标识符名称 比如 ``Identifier not found``。
7. ``TypeError``: 类型系统内的错误，例如无效类型转换，无效赋值等。
8. ``UnimplementedFeatureError``: 编译器当前不支持该功能，但预计将在未来的版本中支持。
9. ``InternalCompilerError``: 在编译器中触发的内部错误——应将此报告为一个issue。
10. ``Exception``: 编译期间的未知失败——应将此报告为一个issue。
11. ``CompilerError``: 编译器堆栈的无效使用——应将此报告为一个issue。
12. ``FatalError``: 未正确处理致命错误——应将此报告为一个issue。
13. ``YulException``: Error during Yul Code generation - this should be reported as an issue.
14. ``Warning``: 警告，不会停止编译，但应尽可能处理。
15. ``Info``: 编译器认为用户可能会发现有用的信息，但并不危险，不一定需要处理。


.. _compiler-tools:

Compiler tools
**************

solidity-upgrade
----------------

``solidity-upgrade`` can help you to semi-automatically upgrade your contracts
to breaking language changes. While it does not and cannot implement all
required changes for every breaking release, it still supports the ones, that
would need plenty of repetitive manual adjustments otherwise.

.. note::

    ``solidity-upgrade`` carries out a large part of the work, but your
    contracts will most likely need further manual adjustments. We recommend
    using a version control system for your files. This helps reviewing and
    eventually rolling back the changes made.

.. warning::

    ``solidity-upgrade`` is not considered to be complete or free from bugs, so
    please use with care.

How it Works
~~~~~~~~~~~~

You can pass (a) Solidity source file(s) to ``solidity-upgrade [files]``. If these make use of ``import`` statement which refer to files outside the
current source file's directory, you need to specify directories that are allowed to read and import files from, by passing ``--allow-paths [directory]``. You can ignore missing files by passing ``--ignore-missing``.

``solidity-upgrade`` is based on ``libsolidity`` and can parse, compile and
analyse your source files, and might find applicable source upgrades in them.

Source upgrades are considered to be small textual changes to your source code.
They are applied to an in-memory representation of the source files
given. The corresponding source file is updated by default, but you can pass
``--dry-run`` to simulate to whole upgrade process without writing to any file.

The upgrade process itself has two phases. In the first phase source files are
parsed, and since it is not possible to upgrade source code on that level,
errors are collected and can be logged by passing ``--verbose``. No source
upgrades available at this point.

In the second phase, all sources are compiled and all activated upgrade analysis
modules are run alongside compilation. By default, all available modules are
activated. Please read the documentation on
:ref:`available modules <upgrade-modules>` for further details.


This can result in compilation errors that may
be fixed by source upgrades. If no errors occur, no source upgrades are being
reported and you're done.
If errors occur and some upgrade module reported a source upgrade, the first
reported one gets applied and compilation is triggered again for all given
source files. The previous step is repeated as long as source upgrades are
reported. If errors still occur, you can log them by passing ``--verbose``.
If no errors occur, your contracts are up to date and can be compiled with
the latest version of the compiler.

.. _upgrade-modules:

Available upgrade modules
~~~~~~~~~~~~~~~~~~~~~~~~~~
+----------------------------+---------+--------------------------------------------------+
| Module                     | Version | Description                                      |
+============================+=========+==================================================+
| ``constructor``            | 0.5.0   | Constructors must now be defined using the       |
|                            |         | ``constructor`` keyword.                         |
+----------------------------+---------+--------------------------------------------------+
| ``visibility``             | 0.5.0   | Explicit function visibility is now mandatory,   |
|                            |         | defaults to ``public``.                          |
+----------------------------+---------+--------------------------------------------------+
| ``abstract``               | 0.6.0   | The keyword ``abstract`` has to be used if a     |
|                            |         | contract does not implement all its functions.   |
+----------------------------+---------+--------------------------------------------------+
| ``virtual``                | 0.6.0   | Functions without implementation outside an      |
|                            |         | interface have to be marked ``virtual``.         |
+----------------------------+---------+--------------------------------------------------+
| ``override``               | 0.6.0   | When overriding a function or modifier, the new  |
|                            |         | keyword ``override`` must be used.               |
+----------------------------+---------+--------------------------------------------------+
| ``dotsyntax``              | 0.7.0   | The following syntax is deprecated:              |
|                            |         | ``f.gas(...)()``, ``f.value(...)()`` and         |
|                            |         | ``(new C).value(...)()``. Replace these calls by |
|                            |         | ``f{gas: ..., value: ...}()`` and                |
|                            |         | ``(new C){value: ...}()``.                       |
+----------------------------+---------+--------------------------------------------------+
| ``now``                    | 0.7.0   | The ``now`` keyword is deprecated. Use           |
|                            |         | ``block.timestamp`` instead.                     |
+----------------------------+---------+--------------------------------------------------+
| ``constructor-visibility`` | 0.7.0   | Removes visibility of constructors.              |
|                            |         |                                                  |
+----------------------------+---------+--------------------------------------------------+

Please read :doc:`0.5.0 release notes <050-breaking-changes>`,
:doc:`0.6.0 release notes <060-breaking-changes>` and
:doc:`0.7.0 release notes <070-breaking-changes>`  and :doc:`0.8.0 release notes <080-breaking-changes>` for further details.

Synopsis
~~~~~~~~

.. code-block:: none

    Usage: solidity-upgrade [options] contract.sol

    Allowed options:
        --help               Show help message and exit.
        --version            Show version and exit.
        --allow-paths path(s)
                             Allow a given path for imports. A list of paths can be
                             supplied by separating them with a comma.
        --ignore-missing     Ignore missing files.
        --modules module(s)  Only activate a specific upgrade module. A list of
                             modules can be supplied by separating them with a comma.
        --dry-run            Apply changes in-memory only and don't write to input
                             file.
        --verbose            Print logs, errors and changes. Shortens output of
                             upgrade patches.
        --unsafe             Accept *unsafe* changes.



Bug Reports / Feature Requests
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If you found a bug or if you have a feature request, please
`file an issue <https://github.com/ethereum/solidity/issues/new/choose>`_ on Github.


Example
~~~~~~~

Assume that you have the following contract in ``Source.sol``:

.. code-block:: Solidity

    pragma solidity >=0.6.0 <0.6.4;
    // This will not compile after 0.7.0
    // SPDX-License-Identifier: GPL-3.0
    contract C {
        // FIXME: remove constructor visibility and make the contract abstract
        constructor() internal {}
    }

    contract D {
        uint time;

        function f() public payable {
            // FIXME: change now to block.timestamp
            time = now;
        }
    }

    contract E {
        D d;

        // FIXME: remove constructor visibility
        constructor() public {}

        function g() public {
            // FIXME: change .value(5) =>  {value: 5}
            d.f.value(5)();
        }
    }



Required Changes
^^^^^^^^^^^^^^^^

The above contract will not compile starting from 0.7.0. To bring the contract up to date with the
current Solidity version, the following upgrade modules have to be executed:
``constructor-visibility``, ``now`` and ``dotsyntax``. Please read the documentation on
:ref:`available modules <upgrade-modules>` for further details.


Running the upgrade
^^^^^^^^^^^^^^^^^^^

It is recommended to explicitly specify the upgrade modules by using ``--modules`` argument.

.. code-block:: bash

    solidity-upgrade --modules constructor-visibility,now,dotsyntax Source.sol

The command above applies all changes as shown below. Please review them carefully (the pragmas will
have to be updated manually.)

.. code-block:: Solidity

    pragma solidity ^0.7.0;
    // SPDX-License-Identifier: GPL-3.0
    abstract contract C {
        // FIXME: remove constructor visibility and make the contract abstract
        constructor() {}
    }

    contract D {
        uint time;

        function f() public payable {
            // FIXME: change now to block.timestamp
            time = block.timestamp;
        }
    }

    contract E {
        D d;

        // FIXME: remove constructor visibility
        constructor() {}

        function g() public {
            // FIXME: change .value(5) =>  {value: 5}
            d.f{value: 5}();
        }
    }
