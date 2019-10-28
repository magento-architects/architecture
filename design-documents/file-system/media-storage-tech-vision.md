# Media Storage technical Vision

Media Storage is used for storing files uploaded by an admin user.
Such files include:
- Images. For example, for CMS blocks, or for products.
- Import/export files.

The media files must be shared between all web nodes in case of multi-node setup, so that each node works with the same data.

Media storage is responsible for storing and access of media files, and not for other manipulations with it (for example, image resizing).
See [Image Resizing Technical Vision](file-system/image-resizing-technical-vision.md) for details.

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
Magento Framework provides a wrapper on top of it to increase control over the API, as well as allow future upgrades of the library guaranteeing preservation of Backward Compatibility.   

Specific storage is configured in deployment configuration by a store owner.
Magento application provides list of required storage "areas" ("groups").
Each Magento module that requires media storage MUST declare storage it needs. Default configuration in local filesystem MUST be included.
Store owner can configuring specific storage group to use non-local storage. This might require to specify credentials. 

No image resizing MUST be happening on the fly (during upload or download of images).
Image resizing is not a responsibility of the Media Storage.
See [Image Resizing Technical Vision](file-system/image-resizing-technical-vision.md) for details.

### With CDN

CDN is configured to work with the centralized storage as origin.
Magento is configured with Media Base URL pointing to the CDN.

When an image is uploaded, it's stored to the centralized storage and further processed to be served to the user via CDN.

TBD:
- add diagram of media file transitions (Magento Admin -> Storage -> CDN (pull CDN requests from Storage) -> FE client)
- list supported and not supported combinations of popular storage types and CDNs

### Without CDN

This is suitable for import scenarios, where files must not be served by CDN.
No additional configuration is needed from the store owner or admin, except configuring specific storage group with the necessary storage type.
