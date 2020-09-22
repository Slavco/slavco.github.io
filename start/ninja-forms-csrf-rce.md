# Ninja Forms before 3.4.27.1 simple CSRF to RCE

The bug is more than simple, GET request with Administrator cookies in it, will result with any plugin from WP dot org installation & activation. If we talk about reguar web application, then this won't be even a issue and server missconfiguration could be blamed (writable executable files), but we talk about WordPress where those conditions are advertised as good and are needed in the name of the security :/
 
Today the CSRF issues aren't the thing that much, because browsers protection, developers experience and web admins experience. Anyway, simple GET request cross domain or on same domain that will result with RCE is more than dangerous scenario, easy to be performed and hard to be noticed even by most experienced web masters.

## Eli5 PoC

In `services/bootstrap.php` there is ajax action called `nf_services_install`. If we look at its code we will notice that there isn't CSRF protection in the form of nonce check / validation, but there is only logged in user permissions check (that is why CSRF attack will work against high privileged accounts) 
```
if ( ! current_user_can('install_plugins') )
	die( json_encode( [ 'error' => esc_html__( 'Sorry, you are not allowed to install plugins on this site.' ) ] ) );
	
$plugin = \WPN_Helper::sanitize_text_field($_REQUEST['plugin']);
$install_path = \WPN_Helper::sanitize_text_field($_REQUEST['install_path']);
```
the rest of the code is plugin download based on `$_REQUEST['plugin']` and plugin activation, based on `$_REQUEST['install_path']`. So the final link that need to be opened with `Administrator` cookies in it will be something like this:
```
hxxts://local.target/wp-admin/admin-ajax.php?action=nf_services_install&plugin=coditor&install_path=coditor/coditor.php 
```
Could be shortened link, could be img tag in email or wp instance content, ... There is huge room for sucesfull attack :)

### What is coditor ?
Coditor is WP plugin not updated several years, but still active ( was and after my report isn't any more ) and was vulnerable towards RCE from lowest possible user role on any WP instance e.g. any logged in user. That was possible because: 
```
add_action('wp_ajax_coditor_process_ajax', array($this, 'coditor_process_ajax'));
```
and `coditor_process_ajax` method allows editing and deletion of any file under 
`$coditor = new Coditor(WP_CONTENT_DIR);` => any plugin / theme.

## Few facts

- In 30 minutes I found several forgotten plugins in the repository with RCE in them (preauth or subscriber user role)
- Attackers could deploy their own plugin towards wp dot org repository and to add backdoor in next iterattions
- If you search and you find for complete attack scenario with vulnerable plugin from the repository, then try to report that plugin towards autors.

## Remediation

- Do not allow your plugins / themes directory to be writable by `www-data` user beside the fact it is advertised like that by WP folks
- Update to the latest version of Ninja Forms plugin :/
 
 



  


