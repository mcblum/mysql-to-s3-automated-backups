What
======
This script will automatically back up all MySql databases on your Linux machine.

Why
======
Because you should have backups and databases are, relatively speaking, so small that there's no reason not to. 

Which Technologies
======
AWS command line tools, MySql and Cron.

How
======
Ssh into your machine and become the root user:
```bash
ssh username@ipaddress.of.machine -p port_you_set_default_is_22
sudo su
```

We'll need to install s3cmd for this to work, so do the following:

Ubuntu
```bash
apt-get install s3cmd
```

CentOS 6
```bash
yum install s3cmd
```

CentOS 7
```bash
wget http://ufpr.dl.sourceforge.net/project/s3tools/s3cmd/1.6.1/s3cmd-1.6.1.tar.gz
tar xzf s3cmd-1.6.1.tar.gz
cd s3cmd-1.6.1
sudo python setup.py install
```

Now that that's done, you'll want to sign in to your Amazon Web Services account and select the service `IAM`. Create a new IAM user and select "Programmatic Access". When it asks your for permissions, select "Attach inline policy" in the lower right and use this policy:
 
 ```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "ListAllAtRoot",
            "Effect": "Allow",
            "Action": [
                "s3:ListAllMyBuckets"
            ],
            "Resource": [
                "arn:aws:s3:::*"
            ]
        },
        {
            "Sid": "AccessOnlyNecessaryBucket",
            "Effect": "Allow",
            "Action": [
                "s3:GetBucketLocation",
                "s3:GetObject",
                "s3:ListBucket",
                "s3:PutObject",
                "s3:PutObjectAcl"
            ],
            "Resource": [
                "arn:aws:s3:::bucket-name/*",
                "arn:aws:s3:::bucket-name"
            ]
        }
    ]
}
```
You'll need to change `bucket-name` to the actual name of the bucket in which you want to store your backups. Save this and it should take you to a screen where it gives you your credentials. You should transfer these directly into the server without writing them down. If you need to re-input them, you can always just create a new access key. In order to get these into your server go back to your terminal and type:

```bash
s3cmd --configure
```
It will ask you a series of questions, include your default region. Since you're in the AWS console just look at the url and it should say something like `us-east-1`. That's what you'll want to put. Follow those steps until it says "Your keys are set up correctly". Note: the option where it asks you to save actually has "no" as the default, so make sure you type "y" and hit enter.

Now that you've configured s3cmd, choose a directory where you want to store your script. I like `/opt/scripts` but something like `/usr/bin` is probably more common. For this tutorial we'll use mine:
```bash
cd /opt
mkdir scripts
```
Create an empty file
```bash
touch mysql-backups.sh
```
Use your favorite text editor and open the file:
```bash
vim mysql-backups.sh
```
Paste the following script:
```bash
#!/usr/bin/env bash

# Add your backup dir location, s3 location, password, mysql location and mysqldump location
DATE=$(date +%d-%m-%Y-%T)
BACKUP_DIR="/backup"
S3_BUCKET_NAME=your-s3-bucket-name
S3_DIRECTORY_NAME=the-directory-you-want
DAYS_TO_KEEP=7
MYSQL_USER=root
MYSQL_PASSWORD=your_mysql_root_password
MYSQL=/usr/bin/mysql
MYSQLDUMP=/usr/bin/mysqldump

# purge old backups
find $BACKUP_DIR -mindepth 1 -mtime +$DAYS_TO_KEEP -type f -delete

# To create a new directory into backup directory location
mkdir -p $BACKUP_DIR/$DATE

# get a list of databases
databases=`$MYSQL -u$MYSQL_USER -p$MYSQL_PASSWORD -e "SHOW DATABASES;" | grep -Ev "(Database|information_schema)"`

# dump each database in separate name
for db in $databases; do
echo $db
$MYSQLDUMP --force --opt --user=$MYSQL_USER -p$MYSQL_PASSWORD --skip-lock-tables --databases $db | gzip > "$BACKUP_DIR/$DATE/$db.sql.gz"

done

# Push backups to S3
s3cmd sync -r --delete-removed $BACKUP_DIR/ s3://$S3_BUCKET_NAME/$S3_DIRECTORY_NAME
```
A few things to note about this script. One, make sure you fill out all the stuff in the first block so the script has all of the variables it needs. This script will put the databases in a folder with the date and time in it so there will never be any naming conflicts.


In order to save the file in vim, hit `:` and then `w` and then `enter`. It should save, and then you can do `:q` to quit.

You'll need to make the script excecutable so run `chmod +x ./mysql-backup.sh` before you actually try to run the script. Now let's test it with `./mysql-backup.sh`. If everything goes correctly, it should dump all MySql databases and begin uploading to S3.

Final Thoughts
======
Obviously this script is more valuble if it runs automatically, so change to your Cron directory:
```bash
cd /etc/cron.d
```
and add a cron job:
```bash
vim mysql_hourly_backup
```

When your text editor opens, paste the following:
```bash
0 */1 * * * root /opt/scripts/mysql-backup.sh
```

That will run the script every hour. If you'd like to change the timing, read up on Cron and go to town.

Final Thoughts
======
That's it! Your setup should be complete. Check your S3 the next day just to make sure it's running, and enjoy the piece of mind.

Matt