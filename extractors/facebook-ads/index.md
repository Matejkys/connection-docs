---
title: Facebook Ads
permalink: /extractors/marketing-sales/facebook-ads/
redirect_from:
  - /extractors/facebook-ads/
---

* TOC
{:toc}

Using the [Facebook Marketing API](https://developers.facebook.com/docs/marketing-api/reference/v2.8), 
this extractor allows you to extract ads data from Facebook:
Ad Account [ads](https://developers.facebook.com/docs/marketing-api/reference/adgroup) including adsets, 
campaigns, and also [ads insights](https://developers.facebook.com/docs/marketing-api/reference/adgroup/insights/).

## Create New Configuration
Find Facebook Ads in the **Extractors** section. Create a new configuration and name it. It can be renamed any time.

{: .image-popup}
![Screenshot - Create configuration](/extractors/facebook-ads/createconfig.png)

Authorize the Facebook account with access to the Ad Account you want to extract. 
You will be asked for the `public_profile,ads_management` [permissions](https://developers.facebook.com/docs/facebook-login/permissions). 
You can always revoke the authorization by going to 
[Facebook apps tab](https://www.facebook.com/settings?tab=applications) (under settings) and removing
`Keboola Ads Extractor` from the list.


{: .image-popup}
![Screenshot - Authorize configuration](/extractors/facebook-ads/authorizefb.png)

From the list of fetched Ad Accounts associated with the authorized account, select the Ad Accounts 
you wish to extract.

{: .image-popup}
![Screenshot - Select Facebook Pages](/extractors/facebook-ads/selectadaccounts.png)

## Create New Query
Create a new query and specify what data to extract. If you choose a preconfigured template, 
all necessary fields will fill up automatically.

{: .image-popup}
![Screenshot - New Query](/extractors/facebook-ads/newquery.png)

The query describes the extractor request to be sent to the Facebook Marketing API. Knowing the API will 
make creating a query easy because all options except `name` represent the [Facebook Marketing API 
request](https://developers.facebook.com/docs/graph-api/using-graph-api) parameters.

### Name
The *Name* option describes the query and is used to prefix all table names resulting from the query.
One query can produce multiple tables. If a table name produced by the query matches the query name or its substring trimmed after the last occurrence of an underscore, then the output table name will not be prefixed and the query name will be used instead. 

For example, if the query name is `ads_insights` and the produced table name is `insights`, then the 
output table name will be `ads_insights`. If the query name is `foo` and the produced table name is 
`ads`, then the output table name will be `foo_ads`.

### Endpoint
The *Endpoint* option describes a significant URL part of the request made to the Facebook Marketing API. 
The absolute URL is in the form `https://graph.facebook.com/<api_version>/<endpoint>`. 
For more info see the [list of supported Facebook Marketing API endpoints](https://developers.facebook.com/docs/marketing-api/reference/v2.8). 

A typical example would be [ads](https://developers.facebook.com/docs/marketing-api/reference/adgroup). 
If left empty, then it refers to the data of the Facebook authorized account itself.

### Fields
The *Fields* option describes data returned from the endpoint. Typically, it is a comma-separated list of 
fields but it also can be used to parametrize the fields and nest more endpoints into it. If you look at 
an [ads](https://developers.facebook.com/docs/marketing-api/reference/adgroup) endpoint, it returns all 
created ads. Each ad can contain fields such as `id`, `name`, `adset_id`. The fields parameter in such 
case is `id,name,adset_id`.

- **Fields/Endpoint Nesting**
    Ads can contain recommendations and these can be included in the *fields* as well: `id,name,adset_id,recommendations{title,code,message}`. The comma separated list
    in between the curly brackets `{}` specifies fields of the "nested" field/endpoint
    [ad-recommendation](https://developers.facebook.com/docs/marketing-api/reference/ad-recommendation/). This way more endpoints can be nested and there is no limit of nesting levels.

- **Fields Parametrization**
    Each field can be parametrized by a dot followed by a modifier name and a value in brackets. 
    A typical parameter could be `since`, `until`, `limit`, or any other parameters or modifiers that the particular endpoint offers such as `date_preset` for the [ads insights](https://developers.facebook.com/docs/marketing-api/reference/adgroup/insights/) endpoint.
    The following is an example of parametrized fields: 	`insights.action_breakdowns(action_type).date_preset(last_month).time_increment(1){ad_id,impressions,reach,actions,clicks,spend}`

### Ad Accounts
This option specifies the Ad Accounts that the query will be applied to. Accounts can be chosen from a 
list of selected ad accounts after authorization. The `All Ad Accounts` option means that the query will 
be applied to all selected ad accounts; the `None` option says that a query will not be applied to any Ad
account. This is useful when extracting data about the authorized account itself. The *Accounts* option 
is represented by the Facebook Marketing API parameter `ids` that is a comma separated list of ad account IDs.

### Since and Until (Advanced tab)
The *Since* and *Until* options represent the corresponding Facebook Marketing API request parameters and 
specify a date range that will be applied to time based data retrieved by the **endpoint**. For example, 
if the endpoint is `ads`, then all ads created within the since-until range will be retrieved.

The *Since* or *Until* parameters are parsed via the [strtotime function](http://php.net/manual/en/function.strtotime.php) 
and can be specified:

- **absolutely** -- as a unix timestamp or in `yyyy-mm-dd` format,
- **relatively** -- for instance, `14 days ago` or `last month`.

For consistent results, specify both `since` and `until` parameters. It is also recommended that the time 
range does not exceed 6 months.

### Limit (Advanced Tab)
The *limit* option represents the Facebook Marketing API request parameter `limit`; it is the maximum 
number of objects that may be returned on a single page of the request. (The default is 25, the maximum 
is 100.) It is useful when the Facebook Marketing Api returns an error saying there is too much data 
requested. In such case, you can lower the limit and run the query again.

## Output Data Description
Output data represent a [tree](https://en.wikipedia.org/wiki/Tree_(graph_theory)) where each node is an 
array of objects returned from the Facebook Graph API. The tree is transformed into one or more CSV 
tables. 

Each table row represents one object. Each table has the primary key auto-detected during the 
extraction and so table data is **imported incrementally**. Columns of the output tables represent fields 
from the `Fields` query option. Moreover, each table will always contain the following basic set of columns:

- `ex_account_id` --- Id of the ad account corresponding to the object stored in the row.
- `fb_graph_node` --- Describes the "vertical position" of the object in the resulting tree. For example, for ads it will be `page_ads`, for ads insights it will be `page_ads_insights`.
- `parent_id` ---  Refers to the `id` column of a parent object represented by another row and/or 
table. For instance, if the row is representing an insight object, its parent is ad and `parent_id` is the 
id of the ad. The parent object type can be also determined from the `fb_graph_node` column as a 
substring from the beginning until the last occurrence of the underscore, e.g. `page_ads_insights` -> 
`page_ads`. The top parent id is an ad account id.

### Ads Insights Data Description
Tables containing ads/campaigns insights data have a specific structure. Consider the following query:

- Endpoint parameter: `campaigns`
- Fields parameter: `insights.action_breakdowns(action_type).date_preset(last_28_days).time_increment(1){account_id,account_name,campaign_id,campaign_name,actions}`

 The query asks for [ads action stats](https://developers.facebook.com/docs/marketing-api/reference/ads-action-stats/) data specified by the 
 [insights api field](https://developers.facebook.com/docs/marketing-api/reference/ads-action-stats/) 
 `actions`. In the api response, each insights object contains an array of actions. The resulting table 
 has each insights object copied into rows by the count of all such arrays, i.e., [ads action stats](https://developers.facebook.com/docs/marketing-api/reference/ads-action-stats/) 
 objects listed for each insights object in arrays. Moreover, columns of the resulting insights table 
 contain the column `ads_action_name` with the name of the ads action array (in this case actions) and 
 columns from fields of [ads action stats](https://developers.facebook.com/docs/marketing-api/reference/ads-action-stats/) 
 such as `action_type`, `action_reaction` and `value`.

## Facebook API Version
You can set the version of the Facebook Marketing API that will be applied for all requests made to the 
Facebook Marketing API by the Facebook extractor. Read more about the Marketing API versions 
[here](https://developers.facebook.com/docs/marketing-api/versions).