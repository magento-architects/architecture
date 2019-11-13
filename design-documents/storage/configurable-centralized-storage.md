# Configurable Centralized Storage

## Terms and Background

**Storage** is place for files dynamically changing during application runtime.
In Magento storage is used for such cases as:
- Media files (for Catalog or CMS)
- Import temporary files
- Log files
- Application configuration files in `app/etc`

This proposal covers scenarios for which it is important to have files shared between web nodes, in case of multi-web node setup.
Those are media and import files.
Currently Magento platform supports only local filesystem for storing any kind of files.

Log files can be stored in local file system (on different web nodes) and aggregated afterwards.
Application configuration files should not change during runtime.
This concern should be addresses separately with possibly redesigning the application so that `app/etc` doesn't need to be mutable during application runtime.

Media storage is responsible for storage and access of media files, and not for other manipulations with it (for example, image resizing).
See [Image Resizing Technical Vision](media/image-resizing-technical-vision.md) for details.

## File Sharing between Web Nodes

For scalability, multiple web nodes may be setup to serve clients of Magento application.

The following types of setup should be supported by Magento application.
Store owners should select the option which suits their needs the most.

### Centralized Storage + CDN

Use cases:
- upload/view product images on multi-web node Magento application
- upload/view CMS images/videos on multi-web node Magento application

### Centralized Storage with no CDN

Use case:
- import/export scenario with multi-web node Magento application

For import files, where web access to files is not necessary, while centralized storage is still required.

### Local Filesystem

Use cases:
- development
- small stores, with no necessity for multi-web node setup

In case of single web node used, local filesystem storage is the most simple and efficient storage to be used.
Magento application should provide easy setup, with local filesystem storage, by default.

More performance optimized storage types (e.g., in-memory) can be provided as an option.
But simplicity of setup and low risk is the main criterion for default option.   

## Design

[Flysystem](https://github.com/thephpleague/flysystem) library is used as a file storage adapter.
It is a widely used library that has support for many storage providers, including local, in-memory, AWS S3, Azure, and others.
Magento Framework provides a wrapper on top of it to increase control over the API, as well as allow future upgrades of the library guaranteeing preservation of Backward Compatibility.   

### Storage Configuration

A Magento module declares necessary storage.
Then Magento Framework provides utilities to access this storage.

```xml
<type name="Magento\Framework\Storage\StorageProvider">
    <arguments>
        <argument name="storage" xsi:type="array">
            <item name="media" xsi:type="string">pub/media</item>
        </argument>
    </arguments>
</type>
```

`pub/media` is default location in local filesystem.

Custom storage configuration can be setup in deployment configuration by a store owner.

`env.php`:
```php
    'storage' => [
        'var' => [
            'driver' => 'local',
            'path' => 'var',
        ],
        'media' => [
            'driver' => 's3',
            'creds' => [
                'key'    => 'your-key',
                'secret' => 'your-secret',
            ],
            'region' => 'your-region',
            'version' => 'latest|version',
            'bucket' => 'your-bucket-name',
            'pathPrefix' => 'optional/path/prefix',
        ],
```

Supported drivers are declared in Magento platform (a module or Magento Framework) as factories.

```xml
    <type name="Magento\Framework\Storage\StorageDriversProvider">
        <arguments>
            <argument name="driverFactories" xsi:type="array">
                <!-- In framework -->
                <item xsi:type="object" name="local">League\Flysystem\Adapter\LocalFactory</item>
            </argument>
        </arguments>
    </type>
```
Local driver is always supported and Magento Framework can use it as a fallback.

A module can add another driver:
```xml
    <type name="Magento\Framework\Storage\StorageDriversProvider">
        <arguments>
            <argument name="driverFactories" xsi:type="array">
                <item xsi:type="object" name="s3">My\Module\Storage\Adapter\S3Factory</item>
            </argument>
        </arguments>
    </type>
```

Application flow:

1. Client submits an image to be stored to **media**
2. Application determines that Media functionality requires `media` storage (from the module configuration)
3. Application determines that `media` storage is configured with `s3` driver and specific options
4. Application Maps `s3` driver to `\My\Module\Storage\Adapter\S3Factory` and creates a driver instance using it and previously determined options
5. Application creates an instance of storage object `\Magento\Framework\StorageInterface` (Magento will have single implementation for now) with specific driver
6. Application provides `media` storage as `\Magento\Framework\StorageInterface` instance
7. Media functionality can now use the storage to save the image

### Resizing on Media Files

No image resizing MUST be happening on the fly (during upload or download of images).
Image resizing is not a responsibility of the Media Storage.
See [Image Resizing Technical Vision](media/image-resizing-technical-vision.md) for details.

Admin -> upload product image -> Magento app -> S3 (Magento stores S3 URL for the original image) -> CloudFront
Client <- CloudFront <- S3

Async resize
- resized image may not be available when requested
  - mitigation: provide orig image? provide placeholder? - can be configured

Resize process: fetch image from S3 -> resize (store locally if necessary) -> upload to S3 (what's new location/name?). How Magento knows about resized image URL? Is there a convention to be used originally?
Resizing process should know name of the final (resized) image (either by pattern/convention or specified) as input data, so that Magento app can already rely on the URL for resized image.

### With CDN

CDN is configured to work with the centralized storage as origin.
For example:

- [Amazon S3 + CloudFront](https://aws.amazon.com/blogs/networking-and-content-delivery/amazon-s3-amazon-cloudfront-a-match-made-in-the-cloud/)
- [Azure storage + CDN](https://docs.microsoft.com/en-us/azure/cdn/cdn-create-a-storage-account-with-cdn)
- Local filesystem + CDN (Magento as a CDN origin)

Magento is configured with Media Base URL pointing to the CDN.

When an image is uploaded, it's stored to the centralized storage and further processed to be served to the user via CDN.

Relative image location MUST be the same for any storage type, so that it's possible to switch between CDNs and different storage types without updating relative links to media images:

- local filesystem: "pub/media/catalog/product/a/b/c.jpg"
- AWS S3 "https://my.s3/pub/media/catalog/product/a/b/c.jpg"
- AWS CloudFront "https://my.cdn/pub/media/catalog/product/a/b/c.jpg"

TBD:
- add diagram of media file transitions (Magento Admin -> Storage -> CDN (pull CDN requests from Storage) -> FE client)
- list supported and not supported combinations of popular storage types and CDNs

### Without CDN

This is suitable for import scenarios, where files must not be served by CDN.
No additional configuration is needed from the store owner or admin, except configuring specific storage group with the necessary storage type.

Admin user uploads a file in Admin -> save to storage (e.g., S3) and save URL to DB
Application (e.g., import process) finds the file URL -> fetches the file by the URL from the storage

We'll have to store information about adapter as well. And "URL" can really have different format based on the storage type. Can store as JSON and then parse by the corresponding adapter.

Drawback:
- if import file upload and processing is happening on the same instance, remote storage adds overhead. But in this case, Admin instance is loaded by unnecessary background process (import), so this should not be recommended.
  - in case somebody still wants to process import on the same instance, the option could be to use local filesystem as storage for import files and make sure there is only one Admin instance (or the same instance is used for processing import files) on the infrastructure level

## Resources

- https://aws.amazon.com/what-is-cloud-file-storage/
- https://en.wikipedia.org/wiki/File_system#Types_of_file_systems
- Laravel filesystem - https://laravel.com/docs/5.8/filesystem
- Symfony - https://github.com/thephpleague/flysystem-bundle
- FS implementation in PHP - https://github.com/thephpleague/flysystem (used by both Laravel and Symfony, as well as other frameworks)
   - https://github.com/flagbit/Magento2-Flysystem-S3
- https://aws.amazon.com/blogs/developer/php-application-logging-with-amazon-cloudwatch-logs-and-monolog/
- https://wiki.corp.magento.com/display/MAGE2/Push+CDN
- https://firstsiteguide.com/cdn-guide/
