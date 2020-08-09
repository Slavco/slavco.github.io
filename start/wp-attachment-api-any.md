# WordPress attachment manipulation functions and any post type

Core had put some efforts in order to prevent accessing `attachment` post type functions from another post types. Usually checks are done by calling `get_post` and comparing the post type with `attachment`, but that is selective and only in hand picked places. It is that way because performances mainly, so many functions are lacking this checks.

## Eli5 PoC
One of them, that calls another attachment related is `wp_update_image_subsizes` and there we have the following calls up to `path_join` where we have phar unserialize or `wp_create_image_subsizes` where image editor is in game, but also `getimagesize` is called where phar unserialize is in game again.

* `wp_update_image_subsizes`
	* `wp_get_attachment_metadata`
	* `wp_get_original_image_path`
		* `wp_attachment_is_image`
		* `wp_get_attachment_metadata`
		* `get_attached_file`
		* `path_join`

this function is called in `wp_ajax_media_create_image_subsizes` 
```
function wp_ajax_media_create_image_subsizes() {
	check_ajax_referer( 'media-form' );

	if ( ! current_user_can( 'upload_files' ) ) {
		wp_send_json_error( array( 'message' => __( 'Sorry, you are not allowed to upload files.' ) ) );
	}

	if ( empty( $_POST['attachment_id'] ) ) {
		wp_send_json_error( array( 'message' => __( 'Upload failed. Please reload and try again.' ) ) );
	}

	$attachment_id = (int) $_POST['attachment_id'];

	if ( ! empty( $_POST['_wp_upload_failed_cleanup'] ) ) {
		// Upload failed. Cleanup.
		if ( wp_attachment_is_image( $attachment_id ) && current_user_can( 'delete_post', $attachment_id ) ) {
			$attachment = get_post( $attachment_id );

			// Created at most 10 min ago.
			if ( $attachment && ( time() - strtotime( $attachment->post_date_gmt ) < 600 ) ) {
				wp_delete_attachment( $attachment_id, true );
				wp_send_json_success();
			}
		}
	}

	// Set a custom header with the attachment_id.
	// Used by the browser/client to resume creating image sub-sizes after a PHP fatal error.
	if ( ! headers_sent() ) {
		header( 'X-WP-Upload-Attachment-ID: ' . $attachment_id );
	}

	// This can still be pretty slow and cause timeout or out of memory errors.
	// The js that handles the response would need to also handle HTTP 500 errors.
	wp_update_image_subsizes( $attachment_id );
...
```
where `attachment_id` could be any post `ID` from any `post_type`. So if `author` user role grabs the `_wpnonce` from `wp-admin/media-new.php` could perform the request below with "attacking" post ID. 

```
curl 'http://local.target/wp-admin/admin-ajax.php' -H 'Cookie: ...' --data 'action=media-create-image-subsizes&_ajax_nonce=[media-form-nonce]&attachment_id=[any-post-id-with-protected-atachment-meta]' --compressed
```
How to add protected meta into posts?! Check the facts below. :)

## Few facts

- Yoast Duplicate posts is adding protected meta when cloning if meta / custom field key starts with `\_`
- We had already seen the wordpress-importer in action
- WooCommerce had fixed something similar few months ago

## Remediation

- make sure that `attachment` routines are performed over attachments
- do not allow users to create protected meta


