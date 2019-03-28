---
layout: post
title: 'Change MongoDB data store directory'
date: 2019-03-28 08:30:00 +0700
categories: [database]
tags: [database, mongodb, store]
image: place_holder.jpeg
---

Just move your folder, addd symlink, then tune permission

```
sudo service mongod stop
sudo mv mongodb /new /disk/mongodb
sudo ln -s /new/disk/mongodb/ /var/lib/mongodb
sudo chown mongodb:mongodb /new/disk/mongodb/
sudo service mongod start
```

test if mongodb user can access new location:

```
sudo -u mongodb -s cd /new/disk/mongodb/
```

resolve other permissions issues if necessary

```
sudo usermod -a G <newdisk_grp> mongodb
```
