# WordPress upload any file with image extension 

Big efforts were put into validation and verification of image uploads in WP as a project. Yet, that isn't completed task. 
In the WP core there is function `wp_upload_bits` which allows to be "uploaded" file with any content with any of the allowed extensions.
Beside its widespread usage around eco system https://wpdirectory.net/search/01ED57DFTZ2VWRZCD4W7N80R9A there are two places in the core where it is used e.g. in `xmlrpc` service `mw_newMediaObject` and in `wp_generate_attachment_metadata`. Via xmlrpc there straight forward upload and via attachment metadata "image" creation is possible via faked cover of any allowed audio / video vile. Try it yourself with mp3 and eyeD3 https://eyed3.readthedocs.io/en/latest/

## Eli5 PoC  

```
function upload_any_image(){
    wp_upload_bits("wpdeeply.jpg", "", "Learn wp deeply");    
}

add_action(init, "upload_any_image");
```

## Few facts

- `wp_check_filetype` checks filetype based on extension only
- As default and pushed/advertised `ImageMagic` via `site-health` in the core, this is basically turning WP into webgoat for setups where file uploads are enabled
- XSS attacks on many web setups
- backdoors for max allowed upload file size

## Remediation 

- `add_filter('wp_upload_bits', ...` where bits will be placed in tmp image with specified extension and will be validated against `wp_check_filetype_and_ext`
- check how Woo fixed this in the past :) 

