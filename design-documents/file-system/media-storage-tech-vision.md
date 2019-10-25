# Media Storage technical Vision

## Requirements

Here are requirements of Magento systems to be considered: 

1. Share media files between multiple web nodes
1. Image resizing for catalog (and potentially other) images

### Scenarios

#### Multi-node with CDN

1. Admin creates a product and uploads an image in Admin Panel
1. Image is saved to a centralized storage
1. Image receives a CDN link by which it can be loaded later
1. Front end application (PWA or Magento theme) requests image to display
   1. The requested image should correspond to the size expected in the specific page layout. This can be width x height or alias ("thumbnail", "detailed", etc.)
1. The requested image is returned by CDN to the FE client and displayed ot the user
   1. Pull CDN gets it from the origin (centralized storage)
   1. Push CDN should already have it (pushed on step 2)
   1. Optionally, CDN can be responsible for image resizing 

#### Development Environment: Single Web Node, No CDN

1. Admin creates a product and uploads an image in Admin Panel
1. Image is saved to local file system
1. Image receives a link by which it can be loaded later. For simplicity, it's served by the same server as Magento application and may have the same base URL 
1. Front end application (PWA or Magento theme) requests image to display
   1. The requested image should correspond to the size expected in the specific page layout. This can be width x height or alias ("thumbnail", "detailed", etc.)
1. The requested image is returned by the web server
   1. Image is not resized on the fly

## File Sharing between Web Nodes

The following setups to be supported by Magento application.
Magento users should select the option which suits their needs the most.

### Centralized Storage + CDN

For media storage, a filesystem adapter that supports centralized storage should be used. For example, [Flysystem](https://github.com/thephpleague/flysystem).
Storage is configured in deployment configuration.
CDN is configured to work with the centralized storage as origin.
Magento is configured with Media Base URL pointing to CDN.

When an image is uploaded, it's stored to the centralized storage and further processed to be served to the user via CDN.

TBD:
- add diagram of media file transitions (Magento Admin -> Storage -> CDN (pull CDN requests from Storage) -> FE client)
- list supported and not supported combinations of popular storage types and CDNs

## Resizing

Image resizing is not a responsibility of Magento application.
Images SHOULD NOT be resized by Magento application on the fly:

- neither during upload
- nor during download

Image resizing should be performed asynchronously by a standalone tool.
Possible options:

1. CDN providers. Many (? check) CDN providers allow to specify necessary size in the image URL, and then return resized image.
1. Magento CLI. For simplicity of initial setup of Magento application, Magento should provide CLI tool that resizes images in background.
   1. Currently Magento provides `magento catalog:images:resize` command for catalog images. This command can be used as is or improved for other types of images if necessary.
1. Custom tool. In case neither of the above options satisfy merchant's needs, ability to use custom resizing tool should be provided by Magento.
   1. Option 1: Magento extension. Magento application dispatches a events when an image is uploaded. An extension subscribes to the event and processes as desired.
   1. Option 2: custom tool, no Magento extension. Magento sends a message to which a worker can subscribe and process images.
   1. Option 3: file watcher. Depending on the type of file system used, a custom tool watches changes in it and processes images as they updated.
   
### Front-end Implementation

TBD: discuss current best practices with FE people.

Magento application provides image URLs for FE application to use for loading images.
FE may require different sizes of images based on:
- the position in the layout (purpose of the picture, "thumbnail" vs "detailed")
- and screen size/device used by the visitor

FE application provides context for the image in the URL. For example: https://my.store.com/image/product/1_thumbnail.jpg?w=100&h=100
CDN can serve it.
Magento serves "thumbnail" image of existing size to the client ("w=100&h=100" arguments are not taken into account).
TBD: describe how it should be done ^^
