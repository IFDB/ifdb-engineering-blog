https://engineering.ifdb.com

After creating a new post:

    hugo
    cd public
    aws s3 cp . s3://engineering.ifdb.com --metadata-directive REPLACE --content-type=text/html --cache-control max-age=3600 --recursive --exclude "*" --include "*.html"
