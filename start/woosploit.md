# WooCommerce < 4.1.0 Remote Code Execution

WooCommerce plugin is raw model how good WordPress plugin should look like and very often we can see how many another plugins around eco system very offten borrow knowledge/source from it and that is probably ok. What is important to be mentioned here is the fact that there isn't any log that will show us all of those usages, but changelogs should be enough for developers to get the idea behing Security fixes or not. From the 4.1.0 changelog we have the following:
```
* Security - Fixed unescaped meta data while duplicating products. Reported by Slavco.
```
but is this enough, is everything told and do you get the idea about the vulnerability?! I know, no or maybe is the answer. 

## What is fixed in 4.1.0 

According the logs there is issue with unescaped meta keys while duplicating products and fix is more than obvious e.g. there is `wp_slash` added to the `$meta->key` in `class-wc-data-store-wp.php`.
```
return add_metadata( $this->meta_type, $object->get_id(), wp_slash( $meta->key ), is_string( $meta->value ) ? wp_slash( $meta->value ) : $meta->value, false );

```
So where is the RCE? If we see the code diff experienced eye should get the attention of changes in `class-wc-template-loader.php` file, method `get_template_loader_files` e.g.
```
if ( is_page_template() ) {
			$page_template = get_page_template_slug();

			if ( $page_template ) {
				$validated_file = validate_file( $page_template );
				if ( 0 === $validated_file ) {
					$templates[] = $page_template;
				} else {
					error_log( "WooCommerce: Unable to validate template path: \"$page_template\". Error Code: $validated_file." );
				}
			}
		}
```
and this means that there is added validation to the value returned by `get_page_template_slug` which in fact is `_wp_page_template` protected meta value. This means that `_wp_page_template` could hold any value towards any location and that value to be `included` while rendering templates at WP
```
$template = apply_filters( 'template_include', $template );
	if ( $template ) {
		include $template;
	} elseif
``` 

## Who is affected

Attack surface is bigger than advertised.

- Duplicating products
- Protected meta could be written while importing products trough new Woo importer, remember it is `class-wc-data-store-wp.php`
- Because `class-wc-template-loader.php` any `import` user role is RCE able on Woo setups, but there is contributor attack in game
- Unserialize trough PHAR maybe? 
  
## Remediation

Take measures about content in any uploaded files on your WP setup, but also for the content of the files that could be created while modifing them.

