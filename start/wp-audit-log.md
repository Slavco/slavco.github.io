# WP Activity Log before 4.1.5 unauthenticated SQLi

This is a plugin that allows logging of activities around your wp setup, so wp admins have some insight about what is happening under the hood and what are the activities of different system functionalities and users from different user groups. In some of the classes around code base, specially those related to MySQL could be noticed that there isn't any usage of prepared queries, so we can salute our SQLi. As an extra there is hooking at `query` DB filter, so we have an extra glitch there.

## Eli5 PoC

Plugin allows (I think in premium version) mirroring, archiving and using external MySQL DB as a storage. So bug is simple and straight forward, into classes/Connector/MySQLDB.php every INSERT statement is prone to SQLi, but we are going to explain in short `MirroringAlertsToDB` method. There we have functionality to mirror the event meta ( yup User Agent as an input is there ) and in the code:
```
// Load data Meta from WP.
$meta = new WSAL_Adapters_MySQL_Meta( $_wpdb );
...
$sql      = 'SELECT * FROM ' . $meta->GetTable() . ' WHERE occurrence_id >= ' . $first_occurrence_id;
			$metadata = $_wpdb->get_results( $sql, ARRAY_A );

			if ( ! empty( $metadata ) ) {
				$meta_new = new WSAL_Adapters_MySQL_Meta( $mirroring_db );
				$sql      = 'INSERT INTO ' . $meta_new->GetTable() . ' (occurrence_id, name, value) VALUES ';

				foreach ( $metadata as $entry ) {
					$sql .= '(' . $entry['occurrence_id'] . ', \'' . $entry['name'] . '\', \'' . str_replace( array( "'", "\'" ), "\'", $entry['value'] ) . '\'), ';
				}

				$sql = rtrim( $sql, ', ' );
				$mirroring_db->query( $sql );
			}
```
here we see that data is taken from MySQL DB table and like that is placed in the another `INSERT` query and there we have the following "escaping"
```
str_replace( array( "'", "\'" ), "\'", $entry['value'] )
```
e.g. any `\'` or `\\\'` is classic SQLi entry point towards DB.

## Troll notifications

As I said there is another bug in the plugin e.g. there is hook at `query` filter from the WP DB functionality. In the `classes/Sensors/Database.php` there is:
```
add_action( 'dbdelta_queries', array( $this, 'EventDBDeltaQuery' ) );
add_filter( 'query', array( $this, 'EventDropQuery' ) );
```
and in `EventDropQuery` we have:
```
preg_match( '|DROP TABLE ([^ ]*)|', $query )
...
preg_match( '|CREATE TABLE IF NOT EXISTS ([^ ]*)|', $query )
```
and this means if somewhere in the `$query` there is `DROP` or `CREATE` statement admin will get new shiny notification about table being created. More than good functionality for phishing towards admin.

## Few facts

- Bugs were discovered with mass scaning for vulnerability patterns around wp plugins
- SQLi above could be checked if it is possible to be escalated for stored XSS
- No matter what is the purpose of the plugin never reinvent mission critical functionalities and follow security practices for WP development

## Remediation

- Before you allow some plugin to land and beign activated on your important WP setup, make sure they use prepared DB statements


