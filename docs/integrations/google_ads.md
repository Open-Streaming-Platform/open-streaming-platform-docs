# Introduction

Open Streaming Platform (OSP) is a powerful open-source solution for creating your own video streaming platform. If you're looking to monetize your OSP instance with Google Ads, you'll need to ensure that Google can access your ads.txt file. This article will guide you through the process of modifying the nginx.conf file to add a route for the ads.txt file.

## Prerequisites

- A working OSP installation.
- Root or sudo access to the server where OSP is installed.
- A basic understanding of the Linux command line.
- A text editor, such as nano or vim.
- Step-by-Step Guide

## Backup Your Configuration File

Before making any changes, it's always a good idea to backup your configuration file.

`sudo cp /usr/local/nginx/conf/nginx.conf /usr/local/nginx/conf/nginx.conf.backup`

## Open the nginx.conf File

Use a text editor to open the nginx.conf file. For this example, we'll use nano.

`sudo nano /usr/local/nginx/conf/nginx.conf`

## Modify the Server Block

Scroll through the file until you find the server block that looks like this:

    # NGINX to HTTP Reverse Proxies
    server {
        include /usr/local/nginx/conf/custom/osp-custom-servers.conf;
    
        # set client body size to 16M #
        client_max_body_size 16M;
    
        include /usr/local/nginx/conf/locations/*.conf;
    
        # redirect server error pages to the static page /50x.html
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    
        location /.well-known/acme-challenge {
            root /var/certbot;
        }
    }


Within this block, add the following lines to create a route for the ads.txt file:

    location ~ ^/ads.txt {
        root /opt/osp/static/;
    }

## Save and Close the File

If you're using nano, press CTRL + O to save the file, then press Enter. Press CTRL + X to exit.

## Reload Nginx

After making the changes, you'll need to reload Nginx to apply them.

`
    sudo systemctl reload nginx
`

## Verification

To verify that the route has been added correctly:

- Open a web browser.
- Navigate to http://your-osp-domain/ads.txt.
- You should see the contents of your ads.txt file displayed.

# Conclusion

You've successfully added a route for the ads.txt file in OSP's nginx.conf. This will ensure that Google can access the file, which is a requirement for using Google Ads on your platform. Always remember to backup configuration files before making changes, and test any modifications in a safe environment when possible.
