# WordPress and phar unserialize 

Core team had put some effort into project in order to preven unserialize of user input via `phar` stream wrapper. In general, efforts are good, but 
to excelent there are two more grades and this is the max with this approach. In the core there is function `path_join` that accepts two parameters
`$base` and `$path`. `$path` is then sent towards function `path_is_absolute` and from the code
```
function path_is_absolute( $path ) {
	/*
	 * Check to see if the path is a stream and check to see if its an actual
	 * path or file as realpath() does not support stream wrappers.
	 */
	if ( wp_is_stream( $path ) && ( is_dir( $path ) || is_file( $path ) ) ) {
		return true;
	}
...
```   
it is obvious that `wp_is_stream` will return `TRUE` for `phar://path_here` and will reach `is_dir` and/or `is_file` which will trigger unserialize of user input in WP. 

## Eli5 PoC

In `wp_restore_image` function which could be called by any user with upload rights on WP (author user role by default) there is the following flow:
```
...
$backup_sizes     = get_post_meta( $post_id, '_wp_attachment_backup_sizes', true );
...
if ( isset( $backup_sizes['full-orig'] ) && is_array( $backup_sizes['full-orig'] ) ) {
		$data = $backup_sizes['full-orig'];
...
		$restored_file = path_join( $parts['dirname'], $data['file'] );
		$restored      = update_attached_file( $post_id, $restored_file );

```
There it is, unserialize of user input. How to place custom values in `_wp_attachment_backup_sizes`, check the facts :)

## Few facts

- `import` cabability is fatal for WP because arbitrary post create from low permission user roles
- this is how to meddle with attachment meta
- `path_join` is wide spread accros eco system

# Remediation

- phar vulnerabilities should be handled on stream wrapper level, not with "guarding"
