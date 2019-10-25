# File System Research

File System = FS

## Problems

1. Current FS API is inconvenient
2. Need support for different FS drivers. Including:
   * local FS
   * AWS S3

Take into account that people may use network FS and all FS operations become network operations and may be slow.

## Use Cases

1. Read/write media files. Location: `pub/media`. User-specified location.

Problems with media:
- we process images. This requires images to be available on local FS
    - on loading image (product listing in Admin)




Move to a separate proposal:
2. Write logs. Location: `var/log`, `var/debug`. User-specified location.
3. Write temporary files during import. Location: `var`. User-specified location.
4. Other reads/writes to `var`?

Move to a separate proposal:
5. Read configuration files from all modules or the same module. Example: read all `di.xml` files. Application-specified location.
6. Read configuration from `app/etc/`. Application-specified location?
7. Write to `app/etc`. Example: during installation/upgrade, or when updating cache status, enable/disable modules. Application-specified location.
8. Write generated code to `generated`. Application-specified location.


### Media Gallery Scenarios

1. Upload image when remote FS is not available
   1.1. fallback to local FS and have background process that:
      - tries to upload image to remote FS
      - if successful, update image references
   1.2. accept images and store locally. Put "processing" task in a queue. The task then uploads the image and updates references
   - Is it even possible to find references? Anybody can use media gallery (product, CMS block, etc).
2. Read image when remote FS is not available
   - we want to read images from CDN, so this is the availability issue CDN provider resolves
   - if remote storage is used not for CDN purposes, should we provide ability to cache images locally? If yes, is it per web-node?

### Filesystem Implementations

The most popular PHP implementation for file system is [Flysystem](https://github.com/thephpleague/flysystem).
* It is used in many popular frameworks, such as Symfony, Laravel and others
* It has support for multiple system types by default
* It is extensible, and community implements new adapters to support more filesystem types 

API comparison of Flysystem and our [Push CDN](https://wiki.corp.magento.com/pages/viewpage.action?spaceKey=MAGE2&title=Push+CDN) proposal:

| Feature | Flysystem | Push CDN | Desired API |
| ------ | --------- | --------- | ----------- |
| Check if file exists | `has($path)` | `isExists(path)` | `isExists(path)` |
| Check if path is a file | | `isFile(path)` | `isFile(path)` |
| Check if path is a directory | | `isDirectory(path)` | `isDirectory(path)` |
| Get parent directory | | `getParentDirectory(path)` | ? |
| Create a directory | `createDir($path)` | `createDirectory(path, permissions)` | |
| List directory | `listContents($directory, $recursive)` | `readDirectory(path)` | |
| List directory recursively | `listContents($directory, $recursive)` | `readDirectoryRecursively(path)` | |
| Rename | `rename($from, $to)` | `rename(oldPath, newPath, targetDriver)` | |
| Copy | `copy($from, $to)` | `copy(source, destination, targetDriver)` | |
| Delete a file | `delete($path)`, `readAndDelete($path)` | `deleteFile(path)` | |
| Delete a directory | `deleteDir($path)` | `deleteDirectory(path)` | |
| Get absolute path | | `getAbsolutePath(basePath, path, scheme)` | It's not abstract enough. AWS S3, for example, can't provide any "absolute path". Should URL be provided? |
| Get real path | | `getRealPath(path)` | |
| Get normalized path without existence validation | | `getRealPathSafety(path)` | |
| Get relative path | | `getRelativePath(basePath, path)` | |
| Read file | `read($path)`, `readStream($path)` | `fileGetContents(path, flag, context)` | |
| Write to file | `write($path, $contents)`, `writeStream($path, $resource)`, `update($path, $contents)`, `updateStream($path, $resource)`, `put($path, $contents)`, `putStream($path, $resource)` | `filePutContents(path, content, mode)` | |
| Stat | `getMimetype($path)`, `getTimestamp($path)`, `getSize($path)` | `stat()` | |
| Touch | | `touch(path, modificationTime)` | |
| Check if path is writable | | `isWritable(path)` | |
| Check if file is readable | | `isReadable(path)` | |

## Solution

1. Use Flysystem as implementation
   1.1. Introduce Magento wrapper around `\League\Flysystem\FilesystemInterface` to control API
      1.1.1. Magento wrapper also adds missing functionality (implementation should reuse methods provided by the flysystem, so all drivers are supported by added functionality).
   1.2. Drivers will be used directly from Flysystem. Any new driver should be implemented according to Flysystem rules. We loose control over SPI here. But Magento is an e-commerce system, and not a FS library provider. Drivers won't be exposed anywhere in API.
2. FS drivers
   2.1. **Application** declares which FS drivers are supported in `di.xml` of a corresponding module (or in Magento framework for default `local` FS).
   2.2. Magento declares a driver factory interface (with simple `create(array $config)` method. All 3rd-party drivers should provide a factory that corresponds to this interface.
3. Storage - new concept instead of "directories" or "folders". Storage defines: 1) location, 2) driver (from supported ones), 3) any configuration options required by the driver.
   3.1. **Application** declares list of storages necessary for its work. The declaration includes: name of the storage (e.g., "media") and default location (default driver is "local" used).
   3.2. **SI** declares storage configuration: location, driver, configuration options required by the driver in `env.php`

Declaration of drivers in Magento Framework:
```xml
    <type name="Magento\Framework\FilesystemService\FilesystemServiceProvider">
        <arguments>
            <argument name="driverFactories" xsi:type="array">
                <!-- In framework -->
                <item xsi:type="object" name="local">League\Flysystem\Adapter\LocalFactory</item>
            </argument>
        </arguments>
    </type>
```
Local driver is always supported and Magento Framework can use it as fallback.

A module can add another driver:
```xml
    <type name="Magento\Framework\FilesystemService\FilesystemServiceProvider">
        <arguments>
            <argument name="driverFactories" xsi:type="array">
                <item xsi:type="object" name="s3">My\Cool\Module\Flysystem\Adapter\S3Factory</item>
            </argument>
        </arguments>
    </type>
```

A module can declare a storage it needs with a name and default path in local FS:
```xml
    <type name="Magento\Framework\FilesystemService\FilesystemServiceProvider">
        <arguments>
            <argument name="requiredStorage" xsi:type="array">
                <item name="media" xsi:type="string">pub/media</item>
            </argument>
        </arguments>
    </type>
```

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

Open questions: 

1. How to declare default storage correctly, so SI could configure default storage and then all files will be stored there if no configuration is provided for the storage?
2. Can we move to XML config instead of `env.php`?


What if?
1. Each driver provides XSD for config.
2. SI creates an XML for each driver (include all necessary storages).
This would allow to have a schema for configuration.

### Existing Magento Flysystem Adapters



## Resources

- https://aws.amazon.com/what-is-cloud-file-storage/
- https://en.wikipedia.org/wiki/File_system#Types_of_file_systems
- Laravel filesystem - https://laravel.com/docs/5.8/filesystem
- Symfony - https://github.com/thephpleague/flysystem-bundle
- FS implementation in PHP - https://github.com/thephpleague/flysystem (used by both Laravel and Symfony, as well as other frameworks)
   - https://github.com/flagbit/Magento2-Flysystem-S3
- https://aws.amazon.com/blogs/developer/php-application-logging-with-amazon-cloudwatch-logs-and-monolog/
- https://wiki.corp.magento.com/display/MAGE2/Push+CDN
