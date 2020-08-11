# WordPress protected meta via wp job manager

Adding slashes or removing them is a thing on WP. Most of the meta functions perform that e.g.
they hold that not popular `wp_unslash` against meta keys and values. This means when input towards 
`update_metadata` and `add_metadata` isn't from the current http requst via some web form e.g. html interface there is possibility for user to insert/update protected meta key into post meta database table with simple `\_any-value` meta key format.

## Eli5 PoC

WP Job Manager holds functionality that duplicates Job Post and that is avaliable only via Job Dashboard. That is possible via `job_manager_duplicate_listing` function and there we have the `update_post_meta` call.
```
...
	// phpcs:ignore WordPress.DB.DirectDatabaseQuery.DirectQuery, WordPress.DB.DirectDatabaseQuery.NoCaching -- Easiest way to retrieve raw meta values without filters.
	$post_meta = $wpdb->get_results( $wpdb->prepare( "SELECT meta_key, meta_value FROM {$wpdb->postmeta} WHERE post_id=%d", $post_id ) );

	if ( ! empty( $post_meta ) ) {
		$post_meta = wp_list_pluck( $post_meta, 'meta_value', 'meta_key' );

		$default_duplicate_ignore_keys = [ '_filled', '_featured', '_job_expires', '_job_duration', '_package_id', '_user_package_id' ];
		$duplicate_ignore_keys         = apply_filters( 'job_manager_duplicate_listing_ignore_keys', $default_duplicate_ignore_keys, true );

		foreach ( $post_meta as $meta_key => $meta_value ) {
			if ( in_array( $meta_key, $duplicate_ignore_keys, true ) ) {
				continue;
			}
			update_post_meta( $new_post_id, $meta_key, maybe_unserialize( $meta_value ) );
		}
	}
...
``` 
Everything is fine, but we must have the possibility to insert meta / custom fields into job post. If we check deeply this functionality then this `job_manager_duplicate_listing` is called in `job_dashboard_handler` which is `WP_Job_Manager_Shortcodes` method. Checking its functionlity we have the following:
```
public function job_dashboard_handler() {
	if (
		! empty( $_REQUEST['action'] )
		&& ! empty( $_REQUEST['_wpnonce'] )
		&& wp_verify_nonce( wp_unslash( $_REQUEST['_wpnonce'] ), 'job_manager_my_job_actions' ) // phpcs:ignore WordPress.Security.ValidatedSanitizedInput.InputNotSanitized -- Nonce should not be modified.
	) {

		$action = sanitize_title( wp_unslash( $_REQUEST['action'] ) );
		$job_id = isset( $_REQUEST['job_id'] ) ? absint( $_REQUEST['job_id'] ) : 0;

		try {
			// Get Job.
			$job = get_post( $job_id );

			// Check ownership.
			if ( ! job_manager_user_can_edit_job( $job_id ) ) {
				throw new Exception( __( 'Invalid ID', 'wp-job-manager' ) );
			}

			switch ( $action ) {
...
case 'duplicate':
						if ( ! job_manager_get_permalink( 'submit_job_form' ) ) {
							throw new Exception( __( 'Missing submission page.', 'wp-job-manager' ) );
						}

						$new_job_id = job_manager_duplicate_listing( $job_id );

``` 
From here we see that `_wpnonce` isn't tight up with job id, but also from `job_manager_user_can_edit_job` we see that post type isn't checked, but only capability for user to edit the post and that could be any post from any post type. Having in mind previous knowledge presented on wpdeeply and nature of WordPress, this could be considered as high severity issue. Check the facts for RCE :)

## Few facts

- WP Job Manager isn't alone
- Attacking via core
- What about WooCommerce before 4.1.0 or latest WooCommerce
- Accessing the blocked meta keys in `job_manager_duplicate_listing` via this technique

## Remediation

- :/

