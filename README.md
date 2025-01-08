   npm install -g @artela/aspect-tool
   
   mkdir contract-aspect && cd contract-aspect
   
   aspect-tool init
   
   npm install
   npm run account:create 
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.8.0 <0.9.0;

/**
 * @title Storage
 * @dev Store & retrieve value in a variable
 */
contract Storage {
    address private deployer;

    constructor() {
        deployer = msg.sender;
    }

    function isOwner(address user) external view returns (bool result) {
        if (user == deployer) {
            return true;
        } else {
            return false;
        }
    }

    function getAspectContext(address aspectId, string calldata key) public returns (string memory validationData) {
        bytes memory contextKey = abi.encodePacked(aspectId, key);
        (bool success, bytes memory returnData) = address(0x64).call(contextKey);
        validationData = success ? string(returnData) : '';
    }

    function setAspectContext(string calldata key, string calldata value) public returns (bool) {
        bytes memory contextKey = abi.encode(key, value);
        (bool success,) = address(0x66).call(contextKey);
        return success;
    }
}
mkdir -p ./build/contract

npm run contract:build

## deploy contract
npm run contract:deploy -- --abi ./build/contract/Storage.abi  --bytecode ./build/contract/Storage.bin ...
--contractAccount 0xaa19F4957C890518b577205c41C706F1c07fa0cc --contractAddress 0xFe4b65F17554B45eF7D146B86E030da7A4e250bb
import {
    allocate,
    entryPoint,
    ethereum,
    execute,
    IPostTxExecuteJP,
    IPreTxExecuteJP,
    PostTxExecuteInput,
    PreTxExecuteInput,
    sys, BytesData,
    uint8ArrayToHex,
} from '@artela/aspect-libs';
import {Protobuf} from "as-proto/assembly";

class StoreAspect
    implements IPostTxExecuteJP, IPreTxExecuteJP {
    isOwner(sender: Uint8Array): bool {
        return true
    }

    preTxExecute(input: PreTxExecuteInput): void {
        //for smart contract call
        sys.aspect.transientStorage.get<string>('ToContract').set<string>('HelloWorld');
    }

    postTxExecute(input: PostTxExecuteInput): void {
        const to = uint8ArrayToHex(input.tx!.to);
        let txData = sys.hostApi.runtimeContext.get("tx.data");
        const txDataPt = Protobuf.decode<BytesData>(txData, BytesData.decode);
        const parentCallMethod = ethereum.parseMethodSig(txDataPt.data);
        const value = sys.aspect.transientStorage.get<string>('ToAspect', to).unwrap();
        // setAspectContext method signature value is `9cf3ef1e`
        if (parentCallMethod == "9cf3ef1e") {
            //'HelloAspect' here is set from smart contract
            sys.require(value == "HelloAspect", "failed to get value by contract setting.");
        }
    }

}

// 2.register aspect Instance
const aspect = new StoreAspect();
entryPoint.setAspect(aspect);

// 3.must export it
export {execute, allocate};
