# MView enhancement

## Problem
Magento StoreFront is considering that different that Magento backoffice and MessageBroker can be on different servers and 
communication between them will be done via network. The problem is in big amount of data that should be transported via network.
Here are few examples:

- Customer has 1500 attributes and changes only name of a product. Product ID is added to message queue. As a result
all product information with 1500 attributes is requested instead of only name. This creates networking issues.
- Customer has 100 stores. Customer changes product only on particular store. Product ID is added to message queue. 
Catalog Exporter is requesting product information for all stores, instead of only one. 

This 2 examples are raising networking issue. When big amount of data will be send via network.

## Proposed solutions

### State comparison

Magento keeps entities cache for exporter. In order to quickly send this data to exporter.
In order to send only changed data we can load all product information. Compare this product information with what we have
in cache and send only changed information.

```php
    $data = $this->processor->process($this->feedIndexMetadata->getFeedName(), $indexData);
    $oldData = $this->fetchFeedData(\array_column($updateEntities, $feedIdentity));
    $dataToChange = array_diff($data, $oldData);
```

#### Drawbacks:

- Low performance, because we will still need to load all products information and compare it. We are losing time and resources on loading whole product entity.
Especially when this entity exists for many stores and has many attributes.
- We will stick with cache approach. Because we will need to keep old state in order to compare it with new state. As a result we will need to allocate 
some resources on creating/updating this cache. This make performance even lower.

### Enhanced MView approach

In this approach we propose to record all changes, that happened with an entity.
If only name or price was changed, only one attribute will be sent to message queue. This will allow significantly reduce networking issue.

Was proposed to add additional fields (store_id and attribute_id) to changelog table:

```mysql
CREATE TABLE `catalog_data_exporter_products_cl` (
  `version_id` int unsigned NOT NULL AUTO_INCREMENT COMMENT 'Version ID',
  `entity_id` int unsigned NOT NULL DEFAULT '0' COMMENT 'Entity ID',
  `store_id` text COMMENT 'store_id',
  `attribute_id` text COMMENT 'attribute_id',
  PRIMARY KEY (`version_id`)
) ENGINE=InnoDB AUTO_INCREMENT=15 DEFAULT CHARSET=utf8 COMMENT='catalog_data_exporter_products_cl';
```

Here is how it should looks like on the MView declaration level:

```xml
    <table name="catalog_product_entity_datetime" entity_column="entity_id">
        <additionalColumns>
            <column name="store_id" cl_name="store_id" />
            <column name="attribute_id" cl_name="attribute_id" />
        </additionalColumns>
    </table>
```

Any column can be specified there. `cl_name` is representing column in changelog table. `name` is representing column in subscription table.

On Exporter level we will have array as in example below with comments when and how it should be indexed:

```php
    //All scopes will be indexed
    $indexData = [
        [
            'productId' => 4,
            'scopeId' => 0,
        ],
    ];

  //All scopes will be indexed
    $indexData = [
        [
            'productId' => 4,
        ],
    ];

  //Scope id 2 will be indexed
    $indexData = [
        [
            'productId' => 4,
            'scopeId' => 2,
        ],
    ];


    //Attribute 73 for scope ID 2 will be indexed and send via network.
    $indexData = [
        [
            'productId' => 4,
            'scopeId' => 2,
            'attributeId' => 73
        ],
    ];
```

#### Drawback:

- Need to align this approach with Magento Core.
- There are tables like media_gallery and product_options, when we still need to reindex whole product (alternative approach with static attribute_code) was proposed for this.
- MySQL binding.


