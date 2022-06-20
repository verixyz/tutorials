# 用verixyz的开发示例

## 预先准备
安装Node V14+以及Yarn，Typescript
```bash
mkdir verixyz-demo && cd verixyz-demo
yarn init -y
```
安装开发依赖
```bash
yarn add typescript ts-node --dev
```

安装@verixyz/core和其他功能模块
```bash
yarn add @verixyz/core @verixyz/credential-w3c @verixyz/data-store @verixyz/did-manager @verixyz/did-provider-ethr @verixyz/did-provider-web @verixyz/did-resolver @verixyz/key-manager @verixyz/kms-local  ethr-did-resolver web-did-resolver
```

安装sqlite
```bash
yarn add sqlite
```

## Bootstrap Verixyz 
1. 编写./tsconfig.json配置文件
```js
{
    "compilerOptions": {
    "preserveConstEnums": true,
    "strict": true,
    "target": "es6",
    "rootDir": "./",
    "moduleResolution": "node",
    "esModuleInterop": true,
    "downlevelIteration": true
  }
}
```
2. 创建 ./src/verixyz/setup.ts 文件

```js
//setup.ts bootstrap @verixyz/core和其他模块
// Core interfaces
import { createAgent, IDIDManager, IResolver, IDataStore, IKeyManager } from '@verixyz/core'

// Core identity manager plugin
import { DIDManager } from '@verixyz/did-manager'

// Ethr did identity provider
import { EthrDIDProvider } from '@verixyz/did-provider-ethr'

// Web did identity provider
import { WebDIDProvider } from '@verixyz/did-provider-web'

// Core key manager plugin
import { KeyManager } from '@verixyz/key-manager'

// Custom key management system for RN
import { KeyManagementSystem, SecretBox } from '@verixyz/kms-local'

// Custom resolvers
import { DIDResolverPlugin } from '@verixyz/did-resolver'
import { Resolver } from 'did-resolver'
import { getResolver as ethrDidResolver } from 'ethr-did-resolver'
import { getResolver as webDidResolver } from 'web-did-resolver'

// Storage plugin using TypeOrm
import { Entities, KeyStore, DIDStore, DataStoreORM, PrivateKeyStore, migrations } from '@verixyz/data-store'

// TypeORM is installed with `@verixyz/data-store`
import { createConnection } from 'typeorm'

// This will be the name for the local sqlite database for demo purposes
const DATABASE_FILE = 'database.sqlite'

// You will need to get a project ID from infura https://www.infura.io
const INFURA_PROJECT_ID = '09b8e827d5d5439ca2b0cc795e06d440' //it is jerry.zhang's key, please change your.

// This will be the secret key for the KMS
const KMS_SECRET_KEY = '7782ce53bb2c41342b30f0ca15df8e45188e4046094c06cb46179efde1d1b67b' 
// you can generate a key by running `npx @veramo/cli config create-secret-key` in a terminal


const dbConnection = createConnection({
    type: 'sqlite',
    database: DATABASE_FILE,
    synchronize: false,
    migrations,
    migrationsRun: true,
    logging: ['error', 'info', 'warn'],
    entities: Entities,
  })


  export const agent = createAgent<IDIDManager & IKeyManager & IDataStore & DataStoreORM & IResolver>({
    plugins: [
      new KeyManager({
        store: new KeyStore(dbConnection),
        kms: {
          local: new KeyManagementSystem(new PrivateKeyStore(dbConnection, new SecretBox(KMS_SECRET_KEY))),
        },
      }),
      new DIDManager({
        store: new DIDStore(dbConnection),
        defaultProvider: 'did:ethr:rinkeby',
        providers: {
          'did:ethr:rinkeby': new EthrDIDProvider({
            defaultKms: 'local',
            network: 'rinkeby',
            rpcUrl: 'https://rinkeby.infura.io/v3/' + INFURA_PROJECT_ID,
          }),
          'did:web': new WebDIDProvider({
            defaultKms: 'local',
          }),
        },
      }),
      new DIDResolverPlugin({
        resolver: new Resolver({
          ...ethrDidResolver({ infuraProjectId: INFURA_PROJECT_ID }),
          ...webDidResolver(),
        }),
      }),
    ],
  })

```

### 编写应用
创建./src/create-indentifiers.ts 和 ./src/list-indentifiers.ts

```js
//create-indentifiers.ts
import { agent } from './verixyz/setup'

async function main() {
  const identity = await agent.didManagerCreate()
  console.log(`New identity created`)
  console.log(identity)
}

main().catch(console.log)
```

```js
//list-idnetifiers.ts
import { agent } from './verixyz/setup'

async function main() {
  const identifiers = await agent.didManagerFind()

  console.log(`There are ${identifiers.length} identifiers`)

  if (identifiers.length > 0) {
    identifiers.map((id: any) => {
      console.log(id)
      console.log('..................')
    })
  }
}

main().catch(console.log)
```

### 添加scripts到package.json文件
```json
{
  
  "scripts": {
    "id:list": "ts-node ./src/list-identifiers",
    "id:create": "ts-node ./src/create-identifiers"
  },
  
}
```

### run
```bash
yarn id:create
```
输出：
```bash
yarn run v1.22.18
$ ts-node ./src/create-identifiers
New identity created
{
  did: 'did:ethr:rinkeby:0x03762c3e1f30bd6c15839cb9519aec7312111db3c30847902e40047823add8e053',
  controllerKeyId: '04762c3e1f30bd6c15839cb9519aec7312111db3c30847902e40047823add8e0532eff4695e1e795d996db076f0c6cc88890ec876ad64cc6ff46e662d5dcc124cd',
  keys: [
    {
      type: 'Secp256k1',
      kid: '04762c3e1f30bd6c15839cb9519aec7312111db3c30847902e40047823add8e0532eff4695e1e795d996db076f0c6cc88890ec876ad64cc6ff46e662d5dcc124cd',
      publicKeyHex: '04762c3e1f30bd6c15839cb9519aec7312111db3c30847902e40047823add8e0532eff4695e1e795d996db076f0c6cc88890ec876ad64cc6ff46e662d5dcc124cd',
      meta: [Object],
      kms: 'local'
    }
  ],
  services: [],
  provider: 'did:ethr:rinkeby'
}
Done in 2.71s.
```

```bash
yarn id:list
```
输出：

```bash
yarn run v1.22.18
$ ts-node ./src/list-identifiers
There are 2 identifiers
Identifier {
  did: 'did:ethr:rinkeby:0x02adf73d530c2528f507e3d54573a83a20de310b6f86eff27e415a94af88cccb49',
  provider: 'did:ethr:rinkeby',
  controllerKeyId: '04adf73d530c2528f507e3d54573a83a20de310b6f86eff27e415a94af88cccb49a6e29eba2df3a9d203bf222f5a96de3b648bdeda692f00c00dfbbb00194061d6',
  keys: [
    Key {
      kid: '04adf73d530c2528f507e3d54573a83a20de310b6f86eff27e415a94af88cccb49a6e29eba2df3a9d203bf222f5a96de3b648bdeda692f00c00dfbbb00194061d6',
      kms: 'local',
      type: 'Secp256k1',
      publicKeyHex: '04adf73d530c2528f507e3d54573a83a20de310b6f86eff27e415a94af88cccb49a6e29eba2df3a9d203bf222f5a96de3b648bdeda692f00c00dfbbb00194061d6',
      meta: [Object]
    }
  ],
  services: []
}
..................
Identifier {
  did: 'did:ethr:rinkeby:0x03762c3e1f30bd6c15839cb9519aec7312111db3c30847902e40047823add8e053',
  provider: 'did:ethr:rinkeby',
  controllerKeyId: '04762c3e1f30bd6c15839cb9519aec7312111db3c30847902e40047823add8e0532eff4695e1e795d996db076f0c6cc88890ec876ad64cc6ff46e662d5dcc124cd',
  keys: [
    Key {
      kid: '04762c3e1f30bd6c15839cb9519aec7312111db3c30847902e40047823add8e0532eff4695e1e795d996db076f0c6cc88890ec876ad64cc6ff46e662d5dcc124cd',
      kms: 'local',
      type: 'Secp256k1',
      publicKeyHex: '04762c3e1f30bd6c15839cb9519aec7312111db3c30847902e40047823add8e0532eff4695e1e795d996db076f0c6cc88890ec876ad64cc6ff46e662d5dcc124cd',
      meta: [Object]
    }
  ],
  services: []
}
..................
Done in 2.65s.
```

