# WP Job Manager permission escalation RCE 

WP Job Manager plugin was sitting vulnerable for some time and attack vectors were avaliable with lowest possible user role. Now in version 1.34.4 some hardening was placed in the form of post_type checks & nonces, but meddling with protected meta failed again. Almost the same way as WordPress core did.

## Eli5 PoC

In `job_manager_duplicate_listing` before patch we had the following code:
```
$post_meta = $wpdb->get_results( $wpdb->prepare( "SELECT meta_key, meta_value FROM {$wpdb->postmeta} WHERE post_id=%d", $post_id ) );
...
foreach ( $post_meta as $meta_key => $meta_value ) {
			if ( in_array( $meta_key, $duplicate_ignore_keys, true ) ) {
				continue;
			}
			update_post_meta( $new_post_id, $meta_key, maybe_unserialize( $meta_value ) );
		}
``` 
And we all know that prefixing meta_key with simple \ will result with insert / update of protected meta_key. Beside this approach and nature of `update_post_meta` there is always option to update value of protected meta_key while updating value of ascii-control-char_meta_key. In order to be prevented situations like that the following is done:
```
foreach ( $post_meta as $meta_key => $meta_value ) {
			$sanitized_key = preg_replace( "/[^\x20-\x7E]/", '', $meta_key );

			if ( in_array( $sanitized_key, $duplicate_ignore_keys, true ) ) {
				continue;
			}

			if ( 1 === preg_match( '/^(_wp_|_oembed_)/', $sanitized_key ) ) {
				continue;
			}

			update_post_meta( $new_post_id, wp_slash( $meta_key ), wp_slash( maybe_unserialize( $meta_value ) ) );
		}
```
### Bypassing duplicate ignore keys check

What we have:
```
$sanitized_key = preg_replace( "/[^\x20-\x7E]/", '', $meta_key );

			if ( in_array( $sanitized_key, $duplicate_ignore_keys, true ) ) {
				continue;
			}
``` 
let we try for 
```
$meta_key = "\x08_ignore_key \x00\x01\x03";
```
result will be 
```
$sanitized_key = "_ignore_key ";
```
and that will fail because strict check into `in_array`. Luckily this payload could be prevented if this ignore meta key is WP protected meta, having the form: _wp_|_oembed_ ... because following check, anotherwise will result with update or insert+update in `update_post_meta`. 
```
if ( 1 === preg_match( '/^(_wp_|_oembed_)/', $sanitized_key ) ) {
				continue;
			}
```
Check the following DB query:
```
UPDATE `wp_postmeta` SET `meta_value`="booFFZ" WHERE `meta_key` = concat('_wp_protected-meta ',char(8),char(1)) 
```
and will be sucesful because space before control characters.

### Bypassing _wp_ | _oembed_ check

It is simple, because MySQL will treat Fullwidth Low Line "＿" (U+FF3F) as "_" e.g. Low Line U+005F. This means that 
```
$sanitized_key = preg_replace( "/[^\x20-\x7E]/", '', $meta_key );
```
will receive value of 
```
$sanitized_key = "wp_test";
```
which will fail on 
```
if ( 1 === preg_match( '/^(_wp_|_oembed_)/', $sanitized_key ) ) {
				continue;
			}
```
but will suceed into database (see the Fullwidth Low Line):
```
UPDATE `wp_postmeta` SET `meta_value`="booFFS" WHERE `meta_key` = '＿wp_protected-meta' 
```

## Few facts 

- MySQL is interesting
- You can't "fix" MySQL behaviour with PHP regex 

## Remediation

- Avoid `update_post_meta` and all another `update_metadata` variations. Use `update_metadata_by_mid` and `add_metadata` instead.
- Changing `update_post_meta` with `add_metadata` in duplicate function should do the job- test for shure and don't forget wp_slash on meta key and value.

