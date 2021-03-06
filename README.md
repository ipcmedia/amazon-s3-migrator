amazon-s3-migrator
------------------

This plugin handles the rest of the site, after the `Amazon S3 and CloudFront` and `Amazon Web Services` plugins have been activated and configured for all new images. Those two handle any new images uploaded to the site, so this plugin will handle the migration of the older images.

These instructions are for multi-site installs of WordPress, but should be easy enough to make work for a single-site install.

Setup
=====

Install this plugin via composer.json and then activate using wp-admin > Plugins. If you're using a multi-site WP installation, then you'll find this plugin under the `Network Admin`.

How to use
==========

The first step is simple, copy all images from the existing site to S3. Assuming that you've set the `Object Path` in the `Amazon S3 and CloudFront` plugin to `wp-content/uploads/` (which replicates the current location of your images), then simply copy all images from your server's `wp-content/uploads` folder to S3 with that prefixed folder path.

The easiest way to do this is to use `s3cmd`. Run the following command to setup s3cmd once it's installed, remembering to set it up to use a proxy if you require:

    s3cmd --configure

Now you have it configured, you can use the following command to copy all files for a site from wordpress to S3. Note the additional headers set. These are for adding caching headers to the image to cache them for 10 years. This is useful as WP tries not to upload duplicate images:

    cd <wordpress-folder>/htdocs/wp-content/uploads
    s3cmd put -r --no-progress sites/2/ s3://testbasket/wp-content/uploads/sites/2/ --add-header="Expires:`date -u +"%a, %d %b %Y %H:%M:%S GMT" --date "10 Years"`" --add-header='Cache-Control:max-age=315360000, public'

If you have a lot of images, then it's worth breaking down by month and year.

    s3cmd put -r --no-progress sites/2/2014/02/ s3://testbasket/wp-content/uploads/sites/2/2014/02/ --add-header="Expires:`date -u +"%a, %d %b %Y %H:%M:%S GMT" --date "10 Years"`" --add-header='Cache-Control:max-age=315360000, public'

After you've finished uploading your files, you need to set all of your images to be public:

    s3cmd setacl s3://testbasket/wp-content/ --acl-public --recursive

With the files copied over, ssh into your server, navigate to your wordpress installation and run the following:

    wp s3 Migrate --url="<site_url>" --path="htdocs" --domain=s3-eu-west-1.amazonaws.com/testbucket --type=all --ignore-meta-keys=amazonS3_info

This code runs on the `blog id`, which is retrieved from the database table `wp_blogs` using the `--url` flag.

The options available are:

`domain=<domain>`
The S3 domain and bucket to replace the images with, e.g. s3-eu-west-1.amazonaws.com/testbucket. This corresponds to the path above of s3://testbasket/wp-content/uploads/sites/2/2014/02/

`batch=<count>`
The number of records to test in one go. This defaults to 1000

`type=<type>`
Only migrate a certain type. This can be `all`, `images`, `posts`, `postmeta`, or `options`

`ignore-meta-keys=<ignore-meta-keys>`
Some post meta data is only for storage and doesn't need to be converted. This takes a csv of meta keys

`prefix=<prefix>`
A prefix for the path, for instance an image that needs to be at the path:
https://s3-eu-west-1.amazonaws.com/inspire-ipcmedia-com/inspirewp/live/wp-content/uploads/sites/2/2012/10/P1070252-e1398158155822-630x418.jpg
will require the domain of "s3-eu-west-1.amazonaws.com/inspire-ipcmedia-com" and the prefix of "inspirewp/live/"
