# WordPress attached file meta

WordPress attachment / media protected meta `_wp_attached_file` is single point of failure for WP. Having access to its value results in RCE as we already show in previous writings non binary safe imagick function, theme directory writing and arbitrary file delete. There are different paths towards `_wp_attached_file`, because at the end of the day is meta value, but also huge effort was put in the past to fix all of the paths towards it - ofc. not even close enough. 

## Eli5 PoC

Let we check the `wp_restore_image` WP function. There we have:
```
$meta             = wp_get_attachment_metadata( $post_id );
$file             = get_attached_file( $post_id );
$backup_sizes     = get_post_meta( $post_id, '_wp_attachment_backup_sizes', true );
```
e.g. `_wp_attachment_metadata` and `_wp_attached_file` are not handled as regular meta values, but `_wp_attachment_backup_sizes` is. Later in the code:
```
$data = $backup_sizes['full-orig'];
...
$restored_file = path_join( $parts['dirname'], $data['file'] );
$restored      = update_attached_file( $post_id, $restored_file );
```
And this show us that anyone with control over the `_wp_attachment_backup_sizes` have access towards `_wp_attached_file`. Hint: wordpress-importer and will be used as one of the show cases for full exploitation demo :)

## Few facts

- preventing access towards protected meta with `current_user_can( 'edit_post_meta',...` or just `is_protected_meta` checks before `update_metadata` and `update_post_meta` is impossible and depends of MySQL configuration
- there are a lot of syncs towards `update_attached_file` around WP eco system
- take care when using `wp_insert_post`

## Remediation
- Why would anyone remediate meta value ?!?!?!

