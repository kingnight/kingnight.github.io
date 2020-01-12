---
title: "Realm数据库schema迁移"
description: "Realm数据库schema迁移"
category: programming
tags: swift, ios, realm
---
# 1.Realm支持的自动迁移特性

- schema 版本号变更
- schema添加新对象
- schema移除对象
- 对象添加属性
- 对象移除属性
- 添加主键
-   移除主键
-   添加索引属性
-   移除索引属性

# 2. 数据库表结构调整需要schemaVersion+1


不论是否需要写迁移代码，如果数据库schema结构修改，需要增加schemaVersion

例如；新增字段后， schemaVersion: 2 需要改为3

`let  configuration = Realm.Configuration(fileURL:url ,`
                                  `schemaVersion: 2,`
                   `deleteRealmIfMigrationNeeded: false)`

# 3.原有数据库表没有主键，新增主键是原有属性，会导致崩溃，需要处理版本迁移


举例：

**版本V1**
```
class DBMVideoFeed: Object {
     @objc dynamic var identifierKey: String?
}
```

**版本V2**
```
class DBMVideoFeed: Object {
    @objc dynamic var identifierKey: String?
    override static func primaryKey() -> String? {
        return "identifierKey"
    }
}
```
**原因：**primaryKey必须要求唯一，在版本V1，由于没有主键，属性名为identifierKey的值没有限制，也就是不保证一定有；

当数据迁移的时候，Realm默认使用缺省值去填充所有缺失的属性，当前例子中就是使用""空字符串填充primaryKey；这样是不允许的，会导致数据库崩溃
```
Fatal error: 'try!' expression unexpectedly raised an error: Error Domain=io.realm Code=1 "Primary key property 'DBMVideoFeed.identifierKey' has duplicate values after migration.
```
**解决：**
```
private static let timelineConfig = Realm.Configuration(
    fileURL: try! Path.inLibrary("hyTimeline.realm"),
    schemaVersion: 2,
    migrationBlock: { migration, oldSchemaVersion in
        if (oldSchemaVersion < 2) {
            var nextID = 0
            migration.enumerateObjects(ofType: DBMVideoFeed.className(), { (oldObject, newObject) in
                    newObject!["identifierKey"] = String(nextID)
                    nextID += 1
            })
        }
    },
    shouldCompactOnLaunch: { (totalBytes, usedBytes) -> Bool in
        // totalBytes refers to the size of the file on disk in bytes (data + free space)
        // usedBytes refers to the number of bytes used by data in the file
        // Compact if the file is over 100MB in size and less than 50% 'used'
        let oneHundredMB = 100 * 1024 * 1024
        return (totalBytes > oneHundredMB) && (Double(usedBytes) / Double(totalBytes)) < 0.5
    })
```
# 4.属性改名

Realm支持自动迁移------添加属性和移除属性，但是如果是对原有属性改名，不写迁移代码会导致Realm认为你是删除了旧属性，同时又增加了新属性，旧属性的数据都会被删除，所以你需要手动处理，明确属性新旧名称之间的关系，这样旧属性的数据会迁移到新属性中。

```
Realm.Configuration.defaultConfiguration = Realm.Configuration(
    schemaVersion: 1,
    migrationBlock: { migration, oldSchemaVersion in
        // We haven't migrated anything yet, so oldSchemaVersion == 0
        if (oldSchemaVersion < 1) {
            // The renaming operation should be done outside of calls to `enumerateObjects(ofType: _:)`.
            migration.renameProperty(onType: Person.className(), from: "yearsSinceBirth", to: "age")
        }
    })
```

# 5.线性迁移

app有三个版本，从旧到新依次是

V1

V2

V3

用户使用时有可能是从V1升级到V2，然后从V2升级到V3；还有可能用户直接从V1升级到V3

**所以在写迁移代码时注意，要写非嵌套的if代码**

**if (oldSchemaVersion < X)**

**确保不论从那个版本升级到最新，都能执行到必须的迁移代码**
```
Realm.Configuration.defaultConfiguration = Realm.Configuration(
    schemaVersion: 1,
    migrationBlock: { migration, oldSchemaVersion in
        // We haven’t migrated anything yet, so oldSchemaVersion == 0
        if (oldSchemaVersion < 1) {
            // The renaming operation should be done outside of calls to `enumerateObjects(ofType: _:)`.
            migration.renameProperty(onType: Person.className(), from: "yearsSinceBirth", to: "age")
        }
        if (oldSchemaVersion < 2) {
  
        }
        if (oldSchemaVersion < 3) {
 
        }
    })
```
# 6.新增属性在迁移代码中需要设置默认值

Realm已知问题bug，参考文档<https://realm.io/docs/objc/latest/#migrations>

> *Note that [default property values](https://realm.io/docs/objc/latest/#default-property-values) aren't applied to new objects or new properties on existing objects during migrations. We consider this to be a bug, and are tracking it as [#1793](https://github.com/realm/realm-cocoa/issues/1793).*

例如增加属性

dynamic var title = "Default"

自动迁移后title值是" "，而非"Default"

所以如果这个默认值对程序逻辑判断有影响，则需要增加迁移代码

# 7. 迁移过程中添加主键不能够立即反映到schema变更中，属性只标记为索引。一旦迁移完成并Realm打开文件，主键才准备好。

# 8. 类的子集限定
Realm 中Configuration的初始化方法
```
public init(fileURL: URL? = default, inMemoryIdentifier: String? = default, syncConfiguration: RealmSwift.SyncConfiguration? = default, encryptionKey: Data? = default, readOnly: Bool = default, schemaVersion: UInt64 = default, migrationBlock: MigrationBlock? = default, deleteRealmIfMigrationNeeded: Bool = default, shouldCompactOnLaunch: ((Int, Int) -> Bool)? = default, objectTypes: [RealmSwift.Object.Type]? = default)
```
其中：
- parameter objectTypes:        The subset of `Object` subclasses persisted in the Realm.

这个参数objectTypes默认是default，也就是所有继承Object的schema表均会存储到当前数据库中；在某些情况下，您可能想要限制某些类只能够存储在指定 Realm 数据库中，同时针对包含不同的schema的数据库写不同的迁移代码。例如，假如有两只团队分别负责应用的不同部分，两者都在内部使用了 Realm 数据库，而又不想协调两个团队之间可能会出现的数据迁移。那么您可以通过设置 Realm.Configuration 的 objectTypes 属性来实现这一点。

举例来说：有如下schema结构
```
class DBChatSession: Object{
      @objc dynamic var one:DBChatUser?
      let atUsers = List<DBMAt>()
}

class DBChatUser: Object {

}

class DBMAt: Object {
    @objc dynamic var i:Int16 = 0
    let u = List<DBMAtPerson>()
}

```
这时你需要DBChatSession，DBMAt，DBChatUser，DBMAtPerson都加入objectTypes数组中
否则会报错
```
*** Terminating app due to uncaught exception 'RLMException', reason: 'Invalid class subset list:
- 'DBChatSession.atUsers' links to class 'DBMAt', which is missing from the list of classes managed by the Realm'
*** First throw call stack:
```