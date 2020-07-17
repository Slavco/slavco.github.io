# WordPress importer and attached file

As we already seen, WordPress project have put a lot of effort to prevent meddling with
`_wp_attached_file` meta value. Part of this effort was put into wordpress-importer plugin as part of the core, but not enough. Few versions back there was possibility to bypass those restrictions with `\` magic WP does.

## Eli5 PoC
 
`class-wp-import.php` method `is_valid_meta_key` we have the following:

```
function is_valid_meta_key( $key ) {
		// skip attachment metadata since we'll regenerate it from scratch
		// skip _edit_lock as not relevant for import
		if ( in_array( $key, array( '_wp_attached_file', '_wp_attachment_metadata', '_edit_lock' ) ) )
			return false;
		return $key;
	}
```
and 
```
add_filter( 'import_post_meta_key', array( $this, 'is_valid_meta_key' ) );
```
Do you see `_wp_attachment_backup_sizes` blocked in the code?

## Few facts

- There are reliable attacks against export-import process from low privileged accounts

## Remediation

- Don't allow wordpress-importer to fall back on the `class WXR_Parser_Regex`

