---
layout: posts
title:  "Better value for money for hard drives"
image: disk_thumbnail.jpg

excerpt: "Let's see how we can get more for less! Commercial external hard
drives are a bit of a scam. We'll discuss how we can build better hard drives
for cheaper"

---

# Better value for money for hard drives

<p align="center">
  <img width="500" src="{{ site.baseurl }}/images/disk/thumbnail_cropped.jpg">
</p>

I recently wanted to get a new external and portable drive. My requirements
were:

- As portable as possible: small, light and self-powered
- 1 TB of storage
- Fast: I need a SSD, not an old mechanical hard drive (HDD)
- Cheap (around the £80 mark, less if possible)

It's also important to note that HDD are relatively fragile (they have moving
mechanical parts) and it's best to avoid them for drives that travel.


## State of the market

### HDDs

A quick search on Amazon will give you plenty of choices for a 1TB drive, like [this
one](https://www.amazon.co.uk/Seagate-Portable-External-Drive-STGX1000400/dp/B07CRG7BBH/ref=sr_1_4?crid=1YAB8CTU79LMZ&dib=eyJ2IjoiMSJ9.tJRqmt_W0RjdJiiEsrtw8YEzdRTq2KM4hM3xogsTrMGeUbRQrxV4qg_QAKvVAEiWrpZ-dsxbx27xur8p4shyiIFInxc0KIP5cQqFxzzgMcLuN-qJjrZ3IPQj-4QZx8M_YrjtSLpKztV6WTnpwtbOiHs62LRJmAr7nNQj2ClC5ZcxtJLUjHmWFU4xa4Oi6sVPVCn_hbhdNKPKaGaeq-1mEg70dLti6YwpObTA7EvALEo.qILUZwhxZEJQm5XG8LJQvj3WCtjIxHGUeA1YbzKRSp4&dib_tag=se&keywords=external%2Bhard%2Bdrive%2B1tb&sprefix=external%2Bhard%2B%2Caps%2C74&sr=8-4&th=1).

You can find them in the range £40-£60. However, these are mechanical
hard drives. Most of them run at 5400 rpm (some at 7200 rpm), and they are
really at the low end of what you can get in terms of transfer speed (your
mileage may vary, but a 5400 rpm HDD will give you between 50 and 100 MB/s
over USB 3.0). They are "fragile" (I have had HDDs failing within one year
when I was travelling a lot). It's best to stear clear of this kind of drive.


### SSDs

There are good options out there, like [this
one (£86)](https://www.amazon.co.uk/SanDisk-Extreme-Portable-1050MB-Dust-Resistant/dp/B08GTYFC37/ref=sr_1_4?crid=11U89Q9HKWDL&dib=eyJ2IjoiMSJ9.RI3sO5cg-XNsjJ7Qb771zAU4V9kT6Vn5gde958N5cRcvpefbWIwu1qxJXyw6_cWo5bsKa6QciGQfu9hWxA_ZOglrYE4hwQnAr3brUEPtMuP3rKnzXuzjTFuZv7s2GSupcXOYrm8l1usMgc0s_-udlJqjhX0hnBDbNCczn3QoCEx64tWridkBlHWhmBpxioLS5G6Hdkr2yuXYI5Ojg3CA1dMNnZmuh9KKOCKSwCxG778.PdfDBxRBZsaZlpAacQwMzNhYNXjSoYy7F41Ar-JSA3w&dib_tag=se&keywords=external%2Bhard%2Bdrive%2B1tb%2Bm2&sprefix=external%2Bhard%2Bdrive%2B1tb%2Bm2%2Caps%2C71&sr=8-4&th=1).
This option meets all my requirements, and it's rated IP65. The speed
(1050 MB/s) might be a bit low, but if you don't want to bother and just
want to get a working drive, this is probably a good option.

---

## Building a drive

The last option above would be a good choice, but I decided to see if I could
build (well, assemble really) my own drive. Because at the end of the day,
an external drive is simply a drive and an enclosure. This would allow me
to reuse the parts by disassembling everything. I can also choose the SSD
I want to put inside the enclosure.

Here are the parts I chose:

- [Beikell M.2 NVME Enclosure](https://www.amazon.co.uk/gp/product/B0BGS3NZ4C/ref=ppx_yo_dt_b_search_asin_title?ie=UTF8&psc=1)
- [Crucial P3 Plus 1TB M.2 PCIe Gen4 NVMe Internal SSD (CT1000P3PSSD801)](https://www.amazon.co.uk/dp/B0BYWB6237?psc=1&ref=ppx_yo2ov_dt_b_product_details)

Total spent: £83.98


### Assembly and formatting

The assembly was easy and took around 5 min. I just sticked the pad of thermic
paste on top of the drive, clipped the drive in the enclosure and closed it.

Since I bought a blanck SSD, I needed to format it. I used Gparted to
create an ext4 partition on the drive (I only mount this disk on Linux,
no need for NTFS). That took 30 seconds.


### Benchmark

So, how fast is this thing? I plugged it into my computer using the USB-C
to USB-C cable that came with the enclosure, and I ran a benchmark with
KDiskMark (it's a Linux equivalent to CrystalDiskMark)

<p align="center">
  <img width="500" src="{{ site.baseurl }}/images/disk/stock_cable.png">
</p>

The results above are disappointing, and a bit fishy. We are far below the
5 GB/s advertised for the NVMe disk. Of course manufacturers always provide
"up to" speeds, but the speeds I measured are an order of magnitude below
what I was expecting. Let's try with another USB-C cable:

<p align="center">
  <img width="500" src="{{ site.baseurl }}/images/disk/second_cable.png">
</p>

That's better! It turns out a cable is not just like any other cable...I
had the same experience with an Ethernet cable once: I thought my internet
connection was bad, but it turned out it was just a low quality Ethernet
cable (they actually come in different categories, from 1 to 8, and are
rated for different speeds).
