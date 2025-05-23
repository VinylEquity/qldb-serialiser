# QLDB Serialise

The QLDB Serialiser is a promise-based Node.js ORM for the AWS QLDB Ledger. It is founded on the idea and partially the structure behind Sequelize. 

QLDB Serialise uses the AWS Node.js drivers. See [AWS Qldb node.js driver](https://www.npmjs.com/package/amazon-qldb-driver-nodejs) for more on this.

[![NPM](https://nodei.co/npm/qldb-serialiser.png?downloads=true&downloadRank=true&stars=true)](https://www.npmjs.com/package/qldb-serialiser)

## Installation

```bash
$ npm install qldb-serialiser
```
or clone this project into your node_modules folder.
## Usage
To use the QLDB Serialser in your code create a connector and ue this connector in your model

Connector (/config/qldb.connect.js)
```javascript
const {qldbConnect, Ledger, DataTypes} = require('qldb-serialiser');

let qldbSettings = {
    maxConcurrentTransactions: 10,
    retryLimit: 4,
    region: process.env.AWS_REGION,
    sslEnabled: true
};
let qldb = new qldbConnect(process.env.QLDB_NAME, qldbSettings);
qldb.getTableNames()
    .then(() => {
        console.log("Database has been connected")
    })
    .catch((err) => {
        console.log("Unable to connect to Database");
    });

module.exports = {
    qldb,
    Ledger,
    DataTypes
}
```

Model example (/models/asset.model.js)
```javascript
const {qldb, Ledger, DataTypes} = require('../config/qldb.connect');

const AssetData = require('./asset_data.model');

const Asset = new Ledger(qldb, 'Assets', {
        assetId: {
            type: DataTypes.STRING,
            allowNull: false,
            primaryKey: true,
            index: true
        },
        assetData: {
            type: DataTypes.LEDGER,
            model: AssetData,
            allowNull: true,
        },
        title: {
            type: DataTypes.STRING,
            allowNull: false,
            index: true
        },
        description: {
            type: DataTypes.STRING,
            allowNull: false,
        },
        owner: {
            type: DataTypes.OBJECT,
            allowNull: false,
            model: {
                individual: {
                    type: DataTypes.ARRAY,
                    model: {
                        person: {
                            type: DataTypes.LEDGER,
                            model: Person,
                            allowNull: false
                        },
                        percentage: {
                            type: DataTypes.NUMBER,
                            allowNull: false
                        }
                    },
                    allowNull: false,
                },
            }
        },
        price: {
            type: DataTypes.NUMBER,
            allowNull: false,
        },
        hideOwner: {
            type: DataTypes.BOOLEAN,
            allowNull: false,
            default: false
        },
        hidePrice: {
            type: DataTypes.BOOLEAN,
            allowNull: false,
            default: false
        }
    },
    {
        timestamps: true
    });

module.exports = Asset;
```
Note that in the above sample nested models are made possible with the usage of the DataTypes.LEDGER.
##### Working with Indexes:
QLDB Supports Indexes. indexes improve query performance for seek operations. [See Optimizing query performance](https://docs.aws.amazon.com/qldb/latest/developerguide/working.optimize.html)

So, We've added a wrapper to create indexes on field by setting ```index: true```

Note that, You can create upto 5 indexes only on top level fields per single table.[See QLDB Docs](https://docs.aws.amazon.com/qldb/latest/developerguide/working.create.html)


When inserting data into the LEDGER type standard JSON is expected. All fieldnames are tested in the same way as any other model.
 ```json
{
	"assetData":{
        "id": "....",
		"name": "....",
		"description": "...."
	},
	"title":"....",
	"description":"...."
}
```
## Using records
The next samples show how to interact with the models to manipulate the data in the QLDB. The next few samples use the Asset model as described above.
The sample functions are indicative. 

### Model synchronization
A model can be synchronized with the database by calling `'AssetModel.sync()'`. It will check if table and defined indexes exists in ledger and creates table and indexes if not existed. Call it only once after model is defined.

### Reading a record
Reads can be done in serveral ways. There are three functions; `'getAll()'`, `'getBy()'` and `'getByPk()'`. The `'getByPk()'` function is a wrapper around the `'getBy()'` function that finds the Primary key of the model and based on that creates the where clause needed by the `'getBy()'` function.
````javascript
    async getAssetById(assetId){
        let assetResult = await Asset.getByPk(assetId);
        if(assetResult) return assetResult;
        return false
    }
```` 
Beware of using the `'getAll()`' function as QLDB at the moment does not support paginated results and will get all the records. This may end up in costly queries being run for a long time.

### Adding or inserting record
The adding of a record is done by passing the JSON containing all fields to the add function in the model. An example of the JSON format can be seen above.

````javascript
    async createAsset(asset){
        let assetResult = await Asset.add(asset);
        if(assetResult) return assetResult;
        return false
    }
````

### Select records using arguments
By using the argument `'fields'` in the argument when selecting using the `'getBy()'` or `'getOneBy()'` only those fields will be returned in the results.
If `'fields'` is omitted it acts like a 'SELECT * FROM ...' and will return all fields.
````javascript
    const args = {
        where: {
          Name: 'Asset1',
          Price: 35,
          isActive: true,
          OR:{
            Color:'Green',
            Weight: 28
          }
        }
      }
    async getAsset(args){
        let assetResult = await Asset.getBy(args);
        if(assetResult) return assetResult;
        return false
    }
````
### Reading, Updating, Deleting Records with DocumentId
Set `'useDocumentId: true'` to read, update, delete records with documentId and add `'documentId'` attribute in where clause.
By using the argument `'documentId'` in  `'where'` object, you can read, update, delete records using QLDB's unique document Identifier.


````javascript
    const args = {
      useDocumentId: true, // Set to true if you're doing operations using documentId
        where: {
          documentId:'LuK62uYWnEG8ygfMChBj6V'
        }
      }
    async getAsset(args){
        let assetResult = await Asset.getBy(args);
        if(assetResult) return assetResult;
        return false
    }
````
### Updating a record
To update a record simply pass the updated fields in a JSON model to the model. The form of the whereClause is the same as for a search action '{ fieldname: fieldValue }'. 
```javascript
    async updateAsset(whereClause, updates){
        let assetResult = await Asset.update({fields: updates, where: whereClause})
        if(assetResult) return assetResult;
        return false

    }
```

### Deleting a record
To delete a record you can simply call the delete function. The flag 'recursive' will also go down the tree of linked LEGER types and also delete those.
When the {recursive: false} is set or when omitted only the first level will be deleted. 
```javascript
    async deleteDocument(docId)  {
        const args = {
           where: { id: docId }, 
           recursive: true 
        }
        let result = await Document.delete(args);
        if(result) return result;
        return false;
    }
```

### Searching for a record
The field name in the where clause in the args can be defined in several ways. When referring to a field name in the table use the name of the field. When referring to a field in an object use the dotted notation. 
When referring to a field in a linked ledger use the name and an object as the search values:

First level fields:
```javascript
    const args = {
       where: { id: docId }, 
    }
```

Second level fields
```javascript
    const args = {
       where: { 'addres.street': streetName }, 
    }
```
  
Search in a linked Ledger  
```javascript
    const args = {
       where: { 
          addres: {
            street: streetName,
            number: '55' 
          }, 
    }
```
### Fetching History of Document
QLDB stores the complete history of every document in a table. You can see all the revisions of document you previously inserted, updated and deleted using ```getHistoryBy()```, ```getHistoryByPk()```, ```getHistoryByDocumentId()``` methods. 

````javascript
    const query = {
        where: {
          Name: 'Asset1',
          Price: 35,
          isActive: true,
          OR:{
            Color:'Green',
            Weight: 28
          }
        }
      }
    async getHistory(args){
        let assetHistory = await Asset.getHistoryBy(args);
        if(assetHistory) return assetHistory;
        return false
    }
````
### Fetching Committed Document With metadata and hash

````javascript
    let args = {
      where: {
        hash: "KGfCsRGs0xhm075l/JuEtvQzT9+Ul5g02h4gaDZLNV0=",
        metadata: {
          version: 27,
          id: "QdxjzbojUtC0pkd80wLw6t",
          txId: "9QcF6tHjDRKCYYuQxYt7zX"
        },
        data: {
          Name: "Asset1",
          Color: "Green"
        }
      }
    };
    async getCommittedDocumentBy(args){
        let committedDocumentRevision = await Asset.getCommittedDocumentBy(args);
        if(committedDocumentRevision) return committedDocumentRevision;
        return false
    }
````

### Operators
Since version 1.1.13 operators have been created on the where clause. The available operators are: 'EQ','NE','IN','NOTIN','GT','GTE','LT','LTE'

```javascript
    const {Operators} = require('qldb-serialiser');
    let args = {
        where: {
            id: userId,
            memberType: [ Operators.NOTIN, [....]]
        }
    };
```

### Fake ordering and fake pagination
PartiQL currently does not support ordering of fields or pagination. In order to be able to have some sort of ordering and pagination functions have been added to order the results and paginate them.
Note that this is only an ordering and pagination AFTER the results come back from the QLDB Ledger. So your result initially will still be a full result with possible 100s of results and then paginated.
**Only the first order element is used if present, the rest are ignored.**
```javascript
    const args = {
        where: {id: userId },
        order: {
            name: 'desc'
        },
        offset: pageOffset,
        limit: pageResults ,
    };
```


## Changes
**version 2.0.11**
* Fix for the update function where boolean value was being converted to string

**version 2.0.10**
* Added model synchronization

**version 2.0.9**
* Adding Support for Read, Update, Delete, History by documentId
* Minor fixes

**version 2.0.8**
* Changed to the release (2.2.0) version of the amazon-qldb-driver-nodejs driver

**version 2.0.7**
* Adding Support for Read,Update, Delete by documentId (Experimental)
* Fixed createdAt/UpdatedAt timestamps on update operation not reflecting correctly

**version 2.0.6**
* Adding OR Operator in SqlWhere, adding WHERE compatibility for boolean data types

**version 2.0.5**
* fetching committed document revision by its metadata and hashes

**version 2.0.4**
* Added fake pagination to history
* Added fake ordering to numbered fields and sorting by date 'createdAt/updatedAt'

**version 2.0.3**
* Bug fixes

**version 2.0.2**
* Added indexes

**version 2.0.1**
* Added getHistoryBy functionality to model, ability to add array of strings ,numbers to array data type without model

**version 2.0.0**
* Changed to the release (2.0.0) version of the amazon-qldb-driver-nodejs driver

**version 1.2.2**
* Downstream merge of additional driver parameters. 

**version 1.2.1**
* Changed to the release (1.0.0) version of the amazon-qldb-driver-nodejs driver

**version 1.2.0**
* Changed Ledger based linking from documentId to primary key of the linked ledger. **This is a breaking change.**
* Extended the where clause to search in linked ledgers

**version 1.1.18**
* Added DataTypes.JSON for unstructured data.
* Merged downstream changes by sitleon

**version 1.1.17**
* Merged the fork from aealth back into the main branch.

**version 1.1.16**
* Minor bugfix on where for numeric values.

**version 1.1.15**
* Added 'fake' ordering and 'fake' pagination
* Added documentation on selecting specific fields.

**version 1.1.14**
* Added a delete function complete with recursiveness.

**version 1.1.13**
* Introduced operators for the where clause
* Moved the DataTypes to a separate file (qldb.datatypes.js)
* Added Operations in separate file (qldb.operators.js)

**version 1.1.7 - 1.1.12**
* Moved to the 1.0.0-rc.1 of the amazon-qldb-driver-nodejs
* Minor bugfixes

**version 1.1.5 & 1.1.6**
* Bugfix in the selection of fields on SELECT statements.
* **Breaking change**: removal of the GetAll function. Since pagination is not supported and the args selector was ambiguous to the GetBy function.

**version 1.1.4**
* Added 'IN' selector to where selection function.

**version 1.1.3**
* Bugfix on the nested array update.

**version 1.1.2**
* Added nested updates for nested ledgers 

**version 1.1.1**
* Added getHistory & getHistoryByPk functions to retrieve all history for a record
* Impoved Query generation
* Updated to use amazon-qldb-driver-nodejs v. 0.1.2-preview.1
* Changed behaviour of nested arrays to enable an array of linked ledger models

**version 1.1.0**
* Added update functionality.
* Added record documentation.
* Removed the logutil and ion TS helpers
* Moved the ionparser to separate helper
* More efficient use of the 'amazon-qldb-driver-nodejs' driver


**version 1.0.2**
* Modified LEDGER portion in base model mapping to allow references to existing documents

**version 1.0.0**
* Added complex models
* Added DataTypes.OBJECT and DataTypes.ARRAY
* Refactoring of SQL Insert creation

**version 0.0.3-beta**
* Nested models
* Added DataTpes.LEDGER
* Cleaner ion to object creation
* Updated dependencies
