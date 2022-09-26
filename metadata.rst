.. include:: glossaries.rst
  
.. _metadata:

#################
合约的元数据
#################

.. index:: metadata, contract verification

Solidity编译器自动生成JSON文件，即合约的元数据，其中包含了当前合约的相关信息。
它可以用于查询编译器版本，所使用的源代码，|ABI| 和 |natspec| 文档，以便更安全地与合约进行交互并验证其源代码。

编译器默认会将元数据文件的 IPFS 哈希值附加到每个合约的字节码末尾（详情请参阅下文），
以便你可以以认证的方式获取该文件，而不必求助于中心化的数据提供者。
其他可选项是使用 Swarm hash 以及不添加元数据hash 到字节码。他们可以通过 :ref:`Standard JSON Interface<compiler-api>` 配置。


你必须将元数据文件发布到 IPFS , Swarm （或其他服务），以便其他人可以访问它。该文件可以通过使用 ``solc --metadata`` 命令和 ``--output-dir`` 参数来生成文件， 没有参数则Metadata会写入标准输出。

元数据包含 IPFS 和 Swarm 对源代码的引用，因此除元数据文件之外，你必须上传所有源代码文件。
对于 IPFS， ``ipfs add`` 返回的CID中包含的hash（不是文件的直接sha2-256值）应与字节码中包含的匹配。


元数据文件具有以下格式。 下面的例子将以人类可读的方式呈现。
正确格式化的元数据应正确使用引号，将空白减少到最小，并对所有对象的键值进行排序以得到唯一的格式。
代码注释当然也是不允许的，这里仅用于解释目的。

.. code-block:: javascript

    {
      // 必选：元数据格式的版本
      "version": "1",
      // 必选：源代码的编程语言，一般会选择规范的“子版本”
      "language": "Solidity",
      // 必选：编译器的细节，内容视语言而定。
      "compiler": {
        // 对 Solidity 来说是必须的：编译器的版本
        "version": "0.8.2+commit.661d1103",
        // 可选： 生成此输出的编译器二进制文件的哈希值
        "keccak256": "0x123..."
      },
      // 必选：编译的源文件／源单位，键值为文件名
      sources:
      {
        "myDirectory/myFile.sol": {
          // 必选：源文件的 keccak256 哈希值
          "keccak256": "0x123...",
          // 必选（除非定义了“content”，详见下文）：
          // 已排序的源文件的URL，URL的协议可以是任意的，但建议使用 IPFS 的URL
          "urls": [ "bzzr://56ab...", "dweb:/ipfs/QmN..."  ]
          // Optional: 在源文件中定义的 SPDX license 标识
          "license": "MIT"
        },
        "mortal": {
          // 必选：源文件的 keccak256 哈希值
          "keccak256": "0x234...",
          // 必选（除非定义了“urls”）： 源文件的字面内容
          "content": "contract mortal is owned { function kill() { if (msg.sender == owner) selfdestruct(owner); } }"
        }
      },
      // 必选：编译器的设置 
      "settings":
      {
        // 对 Solidity 来说是必须的： 导入的重命令 已排序的
        "remappings": [ ":g/dir" ],
        // 可选： 优化器的设置（ enabled 默认设为 false ）
        "optimizer": {
          "enabled": true,
          "runs": 500,
          "details": {
            // peephole defaults to "true"
            "peephole": true,
            // inliner defaults to "true"
            "inliner": true,
            // jumpdestRemover defaults to "true"
            "jumpdestRemover": true,
            "orderLiterals": false,
            "deduplicate": false,
            "cse": false,
            "constantOptimizer": false,
            "yul": true,
            // Optional: Only present if "yul" is "true"
            "yulDetails": {
              "stackAllocation": false,
              "optimizerSteps": "dhfoDgvulfnTUtnIf..."
            }
          }
        }
      },
      "metadata": {
          // Reflects the setting used in the input json, defaults to false
          "useLiteralContent": true,
          // Reflects the setting used in the input json, defaults to "ipfs"
          "bytecodeHash": "ipfs"
        }
        // Required for Solidity: File path and the name of the contract or library this
        // metadata is created for.
        "compilationTarget": {
          "myDirectory/myFile.sol": "MyContract"
        },
        // Required for Solidity: Addresses for libraries used
        "libraries": {
          "MyLib": "0x123123..."
        }
      },
      // 必选：合约的生成信息
      "output":
      {
        // 必选：合约的 ABI 定义t. See "Contract ABI Specification"
        "abi": [ /*...*/ ],
        // Required: NatSpec developer documentation of the contract.
        "devdoc": {
          "version": 1 // NatSpec version
          "kind": "dev",
          // Contents of the @author NatSpec field of the contract
          "author": "John Doe",
          // Contents of the @title NatSpec field of the contract
          "title": "MyERC20: an example ERC20"
          // Contents of the @dev NatSpec field of the contract
          "details": "Interface of the ERC20 standard as defined in the EIP. See https://eips.ethereum.org/EIPS/eip-20 for details",
          "methods": {
            "transfer(address,uint256)": {
              // Contents of the @dev NatSpec field of the method
              "details": "Returns a boolean value indicating whether the operation succeeded. Must be called by the token holder address",
              // Contents of the @param NatSpec fields of the method
              "params": {
                "_value": "The amount tokens to be transferred",
                "_to": "The receiver address"
              }
              // Contents of the @return NatSpec field.
              "returns": {
                // Return var name (here "success") if exists. "_0" as key if return var is unnamed
                "success": "a boolean value indicating whether the operation succeeded"
              }
            }
          },
          "stateVariables": {
            "owner": {
              // Contents of the @dev NatSpec field of the state variable
              "details": "Must be set during contract creation. Can then only be changed by the owner"
            }
          }
          "events": {
             "Transfer(address,address,uint256)": {
               "details": "Emitted when `value` tokens are moved from one account (`from`) toanother (`to`)."
               "params": {
                 "from": "The sender address"
                 "to": "The receiver address"
                 "value": "The token amount"
               }
             }
          }
        },
        // Required: NatSpec user documentation of the contract
        "userdoc": {
          "version": 1 // NatSpec version
          "kind": "user",
          "methods": {
            "transfer(address,uint256)": {
              "notice": "Transfers `_value` tokens to address `_to`"
            }
          },
          "events": {
            "Transfer(address,address,uint256)": {
              "notice": "`_value` tokens have been moved from `from` to `to`"
            }
          }
        }
      }
    }


.. warning::
    由于生成的合约的字节码包含元数据的哈希值，因此对元数据的任何更改都会导致字节码的更改。
    此外，由于元数据包含所有使用的源代码的哈希值，所以任何源代码中的，
    哪怕是一个空格的变化都将导致不同的元数据，并随后产生不同的字节代码。

.. note::
    需注意，上面的 ABI 没有固定的顺序，随编译器的版本而不同。尽管从 Solidity 0.5.12 开始，数组保持了一定的顺序。

.. _encoding-of-the-metadata-hash-in-the-bytecode:

字节码中元数据哈希的编码
=============================================

由于在将来我们可能会支持其他方式来获取元数据文件，
类似 ``{"bzzr0"：<Swarm hash>}`` 的键值对，将会以 `CBOR <https://tools.ietf.org/html/rfc7049>`_ 编码来存储。
由于这种编码的起始位不容易找到，因此添加两个字节来表述其长度，以大端方式编码。
所以，当前版本的Solidity编译器，将以下内容添加到部署的字节码的末尾


.. code-block:: text

    0xa2
    0x64 'i' 'p' 'f' 's' 0x58 0x22 <34 bytes IPFS hash>
    0x64 's' 'o' 'l' 'c' 0x43 <3 byte version encoding>
    0x00 0x33

So in order to retrieve the data, the end of the deployed bytecode can be checked
to match that pattern and use the IPFS hash can be used to retrieve the file(if pinned/published).

Whereas release builds of solc use a 3 byte encoding of the version as shown
above (one byte each for major, minor and patch version number), prerelease builds
will instead use a complete version string including commit hash and build date.

.. note::
  The CBOR mapping can also contain other keys, so it is better to fully
  decode the data instead of relying on it starting with ``0xa264``.
  For example, if any experimental features that affect code generation
  are used, the mapping will also contain ``"experimental": true``.

.. note::
  The compiler currently uses the IPFS hash of the metadata by default, but
  it may also use the bzzr1 hash or some other hash in the future, so do
  not rely on this sequence to start with ``0xa2 0x64 'i' 'p' 'f' 's'``.  We
  might also add additional data to this CBOR structure, so the best option
  is to use a proper CBOR parser.

自动化接口生成和 |natspec| 的使用方法
====================================================

The metadata is used in the following way: A component that wants to interact with a contract (e.g. a wallet) retrieves the code of the contract.

It decodes the CBOR encoded section containing the IPFS/Swarm hash of the metadata file. With that hash, the metadata file is retrieved. That file is JSON-decoded into a structure like above.

The component can then use the ABI to automatically generate a rudimentary user interface for the contract.

Furthermore, the wallet can use the NatSpec user documentation to display a human-readable confirmation message to the user
whenever they interact with the contract, together with requesting authorization for the transaction signature.

有关 |natspec| 的更多信息, read :doc:`Ethereum Natural Language Specification (NatSpec) format <natspec-format>`.


源代码验证的使用方法
==================================

为了验证编译，可以通过元数据文件中的链接从 IPFS/Swarm 中获取源代码。
获取到的源码，会根据元数据中指定的设置，被正确版本的编译器（应该为“官方”编译器之一）所处理。
处理得到的字节码会与创建交易的数据或者 ``CREATE`` 操作码使用的数据进行比较。
这会自动验证元数据，因为它的哈希值是字节码的一部分。
而额外的数据，则是与基于接口进行编码并展示给用户的构造输入数据相符的。

在 `sourcify <https://github.com/ethereum/sourcify>`_ 库 (`npm package <https://www.npmjs.com/package/source-verify>`_) 
可以看到如何使用该特性的示例代码。
