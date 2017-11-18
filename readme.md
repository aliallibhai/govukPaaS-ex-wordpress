## GOV.UK PaaS PHP Example Application:  Wordpress

This is an example application which can be run on GOV.UK PaaS using the [PHP Build Pack].

This is an out-of-the-box implementation of Wordpress 4.0.  It's an example of how common PHP applications can easily be run on GOV.UK PaaS.

### Usage

1. Clone the app (i.e. this repo).

  ```bash
  git clone https://github.com/aliallibhai/govukPaaS-ex-wordpress
  cd govukPaaS-ex-wordpress
  ```

2. Download and unzip the latest version of [WordPress]

3. Move all of the WordPress files into your govukPaaS-ex-wordpress folder. Be careful not to overwrite the '`wp-config.php` and `wp-db.php` files.

4.  If you don't have one already, create a MySQL service.  With GOV.UK PaaS, the following command will create a free MySQL database through MySql. Here we have named the database wpdb, but you can pick a name of your choice.

  ```bash
  cf create-service mysql free wpdb
  ```

5. Edit the manifest.yml file.  Change the 'name' attribute to something unique.  Then under "services:" change "wpdb" to the name of your MySQL service.  This is the name of the service that will be bound to your application and thus used by WordPress.

6. Like every normal Wordpress install, edit `wp-config.php` and change the [secret keys].  These should be uniqe for every installation.  You can generate these using the [WordPress.org secret-key service].

7. Push it to CloudFoundry.

  ```bash
  cf push
  ```

  Access your application URL in the browser.  You'll see the familiar Wordpress install screen.  Setup your password and your all set.


### Persistent Storage

If you've ever used Wordpress before, you're probably familiar with the way that you can install new themes and plugins through the WebUI.  This and other actions like uploading media files work by allowing the Wordpress application itself to modify files on your local disk.  Unfortunately, this is going to cause [a problem](#caution) when you deploy to CloudFoundry.

A potential approach to solving this problem is to  bundle these files, themes, plugins and media, with your application. That way when you `cf push`, the files will continue to exist. However, this means that before the next time you push your app you need to download a download/backup a copy of the above-mentioned files.

There are multiple problems with this approach, like large and possibly slow uploads, tracking what's changed by actions in Wordpress and the fact that you probably don't want to push every time you need to upload media.  Given this, it's likely that for any serious installation of Wordpress you want a better solution.

There is some useful discussion of these solutions [here]

### Changes

These changes were made to prepare Wordpress to run on CloudFoundry.

1. Edit `wp-config.php`, configure to use CloudFoundry database and connect to it over SSL.

```diff
*
  * @package WordPress
  */
+// ** Get MYSQL database info from Cloud Foundry ** //
+ $services = getenv("VCAP_SERVICES");
+ $services_json = json_decode($services,true);
+ $mysql_config = $services_json["mysql"][0]["credentials"];
-// ** MySQL settings - You can get this info from your web host ** //
+// ** Take the information we gathered from Cloud Foundry and define it in the standard WordPress fields ** //
 /** The name of the database for WordPress */
- define('DB_NAME', 'database_name_here');
+ define('DB_NAME', $mysql_config["name"]);

 /** MySQL database username */
-define('DB_USER', 'username_here');
+ define('DB_USER', $mysql_config["username"]);

 /** MySQL database password */
-define('DB_PASSWORD', 'password_here');
+ define('DB_PASSWORD', $mysql_config["password"]);

 /** MySQL hostname */
-define('DB_HOST', 'localhost');
+ define('DB_HOST', $mysql_config["host"]);

+ /** Enabling SSL */
+ define('MYSQL_CLIENT_FLAGS', MYSQL_CLIENT_SSL);
+ define('MYSQL_SSL_CERT', "/etc/ssl/certs/ca-certificates.crt");

 /** Database Charset to use in creating database tables. */
 define('DB_CHARSET', 'utf8');
```

2. Edit `wp-db.php` file, configure to enable SSL connections to the MySQL database.
```diff
+     /*Enable WordPress to connect to Amazon RDS over SSL.
+       * ADDED per https://core.trac.wordpress.org/ticket/28625 */
+     /* call set_ssl if mysql client flag set and settings available*/
+ if ( $client_flags & MYSQL_CLIENT_SSL ) {
+     $pack = array( $this->dbh );
+     $call_set = false;
+     foreach( array( 'MYSQL_SSL_KEY', 'MYSQL_SSL_CERT', 'MYSQL_SSL_CA',
+         'MYSQL_SSL_CAPATH', 'MYSQL_SSL_CIPHER' ) as $opt_key ) {
+         $pack[] = ( defined( $opt_key ) ) ? constant( $opt_key ) : null;
+         $call_set |= defined( $opt_key );
+}
+     /* Now if anything was packed - unpack into the function.
+     * Note this doesn't check if paths exist, as per the PHP doc
+     * at http://www.php.net/manual/en/mysqli.ssl-set.php: "This
+     * function always returns TRUE value. If SSL setup is incorrect
+     * mysqli_real_connect() will return an error ..."
+     */
+     if ( $call_set ) { // SSL added here!
+         call_user_func_array( 'mysqli_ssl_set', $pack );
+     }
}
      if ( WP_DEBUG ) {
        mysqli_real_connect( $this->dbh, $host, $this->dbuser, $this->dbpassword, null, $port, $socket, $client_flags );
      } else {
        @mysqli_real_connect( $this->dbh, $host, $this->dbuser, $this->dbpassword, null, $port, $socket, $client_flags );
      }
````

### Caution

Please read the following before using Wordpress in production on CloudFoundry.

1. Wordpress is designed to write to the local file system.  This does not work well with CloudFoundry, as an application's [local storage on CloudFoundry] is ephemeral.  In other words, Wordpress will write things to the local disk and they will eventually disappear.  See the [Persistent Storage](#persistent-storage) above for ways to work around this.

1. This is not an issue with Wordpress specifically, but PHP stores session information to the local disk.  As mentioned previously, the local disk for an application on CloudFoundry is ephemeral, so it is possible for you to lose session and session data.  If you need reliable session storage, look at storing session data in an SQL database or with a NoSQL service.


[here]: https://github.com/cloudfoundry-samples/cf-ex-wordpress
[WordPress version]: https://wordpress.org/download/
[PHP Build Pack]:https://github.com/cloudfoundry/php-buildpack
[secret keys]:https://github.com/aliallibhai/govukPaaS-ex-wordpress/blob/master/wp-config.php#L49
[WordPress.org secret-key service]:https://api.wordpress.org/secret-key/1.1/salt
[local storage on CloudFoundry]:http://docs.cloudfoundry.org/devguide/deploy-apps/prepare-to-deploy.html#filesystem
[wp-content directory]:http://codex.wordpress.org/Determining_Plugin_and_Content_Directories
[ephemeral file system]:http://docs.cloudfoundry.org/devguide/deploy-apps/prepare-to-deploy.html#filesystem
