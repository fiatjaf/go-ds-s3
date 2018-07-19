An [IPFS](https://ipfs.io) [datastore implementation](https://github.com/ipfs/go-datastore) for Amazon's S3 service.

---

To use, you must modify `go-ipfs`'s [repo/fsrepo/datastores.go](https://github.com/ipfs/go-ipfs/blob/master/repo/fsrepo/datastores.go) like the following:

```go
diff --git a/repo/fsrepo/datastores.go b/repo/fsrepo/datastores.go
index be565de..ca86951 100644
--- a/repo/fsrepo/datastores.go
+++ b/repo/fsrepo/datastores.go
@@ -18,6 +18,8 @@ import (
 	flatfs "gx/ipfs/Qmc1ExJkrEUesbvRNHq6g3bJ2km7K4XffcC2bvwdpfGYDx/go-ds-flatfs"
 	ds "gx/ipfs/QmeiCcJfDW1GJnWUArudsv5rQsihpi4oyddPhdqo3CfX6i/go-datastore"
 	mount "gx/ipfs/QmeiCcJfDW1GJnWUArudsv5rQsihpi4oyddPhdqo3CfX6i/go-datastore/mount"
+
+	s3ds "github.com/qri-io/go-ds-s3"
 )
 
 // ConfigFromMap creates a new datastore config from a map
@@ -62,6 +64,7 @@ func init() {
 		"flatfs":   FlatfsDatastoreConfig,
 		"levelds":  LeveldsDatastoreConfig,
 		"badgerds": BadgerdsDatastoreConfig,
+		"s3ds":     S3DatastoreConfig,
 		"mem":      MemDatastoreConfig,
 		"log":      LogDatastoreConfig,
 		"measure":  MeasureDatastoreConfig,
@@ -94,6 +97,7 @@ type premount struct {
 // MountDatastoreConfig returns a mount DatastoreConfig from a spec
 func MountDatastoreConfig(params map[string]interface{}) (DatastoreConfig, error) {
 	var res mountDatastoreConfig
+
 	mounts, ok := params["mounts"].([]interface{})
 	if !ok {
 		return nil, fmt.Errorf("'mounts' field is missing or not an array")
@@ -410,3 +414,53 @@ func (c *badgerdsDatastoreConfig) Create(path string) (repo.Datastore, error) {
 
 	return badgerds.NewDatastore(p, &defopts)
 }
+
+type s3dsDatastoreConfig struct {
+	region    string
+	accessKey string
+	secretKey string
+	bucket    string
+	domain    string
+}
+
+// S3DatastoreConfig returns a configuration stub for an s3 datastore
+// from the given parameters
+func S3DatastoreConfig(params map[string]interface{}) (DatastoreConfig, error) {
+	var domain string
+	if idomain, ok := params["domain"]; ok {
+		domain = idomain.(string)
+	}
+
+	var region string
+	if iregion, ok := params["region"]; ok {
+		region = iregion.(string)
+	}
+
+	return &s3dsDatastoreConfig{
+		accessKey: params["accessKey"].(string),
+		secretKey: params["secretKey"].(string),
+		bucket:    params["bucket"].(string),
+		domain:    domain,
+		region:    region,
+	}, nil
+}
+
+func (c *s3dsDatastoreConfig) DiskSpec() DiskSpec {
+	return map[string]interface{}{
+		"type":      "s3ds",
+		"domain":    c.domain,
+		"bucket":    c.bucket,
+		"accessKey": c.accessKey,
+		"secretKey": c.secretKey,
+	}
+}
+
+func (c *s3dsDatastoreConfig) Create(path string) (repo.Datastore, error) {
+	ds := s3ds.NewDatastore(c.bucket, func(o *s3ds.Options) {
+		o.Region = c.region
+		o.AccessKey = c.accessKey
+		o.AccessSecret = c.secretKey
+	})
+
+	return ds, nil
+}
```

and then recompile.

When initializing the repo with `ipfs init`, provide a `<default-config>` that contains at least

```json
{
  "mounts": [
    {
      "accessKey": "<s3-access-key>",
      "bucket": "<s3-bucket-name>",
      "mountpoint": "/blocks",
      "secretKey": "<s3-secret-key>",
      "region": "us-east-1",
      "type": "s3ds"
    },
    {
      "child": {
        "compression": "none",
        "path": "datastore",
        "type": "levelds"
      },
      "mountpoint": "/",
      "prefix": "leveldb.datastore",
      "type": "measure"
    }
  ],
  "type": "mount"
}
```
