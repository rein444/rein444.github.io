---
title: 'An Overview of IPFS: Usage, Explanation, and Thoughts'
date: 2021-03-18 10:21:22
tags: [tech]
published: true
hideInList: false
feature: 
isTop: false
---
Until now, the World Wide Web relies on HTTP to transfer data and establish communication across the web. But in 2015, an American guy named Juan Benet created a new protocol called IPFS, or you might like its sci-fi sounding name better, *"InterPlanetary File System"*, to compete with HTTP. Instead of the classic server-client approach, IPFS decides to go with peer-to-peer, which should sound familiar if you know BitTorrent. **To summarize it in a short sentence, IPFS is a peer-to-peer file sharing system that claims to be faster, safer, and more open.**

The goal of IPFS is to provide a better alternative for HTTP and ultimately, to replace HTTP. This sounds extremely ambitious, and that's why it interests me. However, after tried it for a little while, I have to say that IPFS is still too immature compared to HTTP, considering that the later one has dominated the web (to be more precise, it started the web) since its creation in the 90s and has evolved a lot after that. IPFS is a promising and somehow a little revolutionary project, but it still needs time to develop.

____

## Usage

First, let's take a look at how to use IPFS. In order to use IPFS, there are some basic concepts one needs to know:
- **Every file on the IPFS network has a unique hash, known as CID**, that resembles a location address and we can use the hash to point to a specific file.
- **IPFS is decentralized, which makes it function like a CDN**. Files are being kept multiple backups so even if the node who originally published the specific file is down, the requested file can still be retrieved from other nodes. This process ensures speed and availability of most files.
- **IPFS is a self-certifying file system (SFS)**, which means that o special permission is needed for data exchange.
- **The content on IPFS is not permanent unless it is pinned**.

### How to Retrieve Content
There are currently four ways to retrieve content from IPFS: public gateway, local gateway, get/cat command, and Integrated IPFS in Brave browser.

### Public Gateway
Protocol Labs maintains a [list of public gateways](https://ipfs.github.io/public-gateway-checker/) that allows users to view the content of files on the  IPFS network without installing IPFS on their local machines.

To use it, choose an online gateway and follow the syntax here:
`<gateway address>/ipfs/<hash>`

For example, here is what you will type in your browser to access a copy of the Turkish wikipedia:
`ipfs.io/ipfs/QmT5NvUtoM5nWFfrQdVrFtvGfKFmG7AHE8P34isapyhCxX/`

### Local Gateway
Download IPFS-Desktop from [here](https://github.com/ipfs-shipyard/ipfs-desktop/releases/tag/v0.14.0) or use the Go implementation of IPFS by following this [guide](https://docs.ipfs.io/install/command-line/) if you are on a remote server or just think GUI is bloated.

Initialize the repository:
`ipfs init`

Take the node online:
`ipfs daemon`
*(or use a service manager if on Unix systems)*

Now you can access content by typing this in your browser:
`127.0.0.1:8080/ipfs/<hash>`

The port can be specified in the config file.

### get/cat Command
Make sure you have installed IPFS-Desktop or go-ipfs and the daemon is started.

The ipfs cat command is similar to the cat command on Unix systems and it reads content from the IPFS network.
`ipfs cat <hash>`

Alternatively, you can use the get command to download the content.
`ipfs get <hash>`

If nothing returns, it is possible that none of the peers you are connected to holds a copy of the file you requested. To check if any peer has the file, run:
`ipfs dht findprovs <hash>`

You cannot request a file from a specific peer because there is no point in doing this. All the files that share the same hash contain the identical content.

### Integrated IPFS in Brave Browser
Brave anounced their support of IPFS in January this year. You can type this in Brave browser to access content:
`ipfs://<hash>`

Be aware that by default, Brave uses public gateways to load the requested content. Brave will suggest you to enable IPFS local daemon support if you are visiting through a public gateway, and by clicking "Enable IPFS", Brave will automatically download go-ipfs and route all IPFS traffic through your local gateway.

### How To Add Content
Download IPFS-Desktop [here](https://github.com/ipfs-shipyard/ipfs-desktop/releases/tag/v0.14.0) or follow this [guide](https://docs.ipfs.io/install/command-line/) to install go-ipfs if you have not already done so. Now you are able to upload file from the desktop app, webUI, or directly from the command line. I guess [pinata](https://pinata.cloud/) will also work, but I have never tried it.

### Desktop App & GUI
Open the app or access the webUI by typing this in your browser:
`127.0.0.1:5001/webui`
*(you need to start the daemon first in order to access the webUI and the port can be speficied in the config file or through 'settings')*

Click import and pin it if needed.

### Command line
To add a file, run:
`ipfs add path/to/file`

Add -r to add a directory.
`ipfs add -r path/to/directory`
*(Record the hash because you need it later to access your file. )*

To pin the file, run:
`ipfs pin add <hash>`

Or add -r for directory.

Other basic commands are explained by `ipfs -h`. Please use them for reference.

----
## Structure

Now we can get deeper into the structure of IPFS.

### DHT
**DHT, or distributed hash table, is a data structure used by IPFS that stores information as key pairs**. Relying on DHT's decentralization, IPFS is able to function even if some nodes do down.

### Merkle DAG
IPFS uses Merkle DAG as its data structure. According to the [official documentation of IPFS](https://docs.ipfs.io/concepts/merkle-dag/#further-resources), *"A Merkle DAG is a DAG where each node has an identifier, and this is the result of hashing the node's contents — any opaque payload carried by the node and the list of identifiers of its children — using a cryptographic hash function like SHA256. "* Merkle DAG resembles a "reversed tree" in which grows from leaves to branch. In order to link children's identifiers to the parents, children's identifiers must be calculated first.

If you recall how I mentioned why request to a specific peer is unnecessary, here is a more detailed explanation: **all nodes are given a unique CID (identifier or can just be called hash) to be identified from the network and the CID is linked to all its descendants**. Copies of the requested objects with that CID from all peers are identical because they share the same DAG. If a change is made in one node, its identifier would alter which also causes its ascendants to alter correspondingly. This process creates a new DAG that cannot be retrieved using the original identifier.

### IPFS Objects
An IPFS object is a structure that contains two fields, which can be visualized as follows
- data: unstructured binary data < 256kb
- Links: links that point to other IPFS objects
  - Name: the name of the link
  - Hash: the hash of the linked object

You can see the structure of any object by running `ipfs object get <hash>`.

If the object is smaller than 256kb, the *links* section will be empty because there is no other sectors to point to. But if the object is larger than 256kb, it is going to be further divided into sectors that are smaller than 256kb and the *name* section will be empty. Each sector will get an unique hash and the final hash for the file will be calculated after combining all the sector hashes into one group. This works inversely when retrieving files. When an user requests a file from the network by hash, the sector hashes specified by that file hash will be searched and downloaded from the node that holds it. After that, they will be arranged in the correct order and will be assembled locally.

IPFS also works like Git that it supports versions. The commit object will have one or more links that point to previous commits, and one object link that points to the DAG for that commit.

----

Even though the concept of a decentralized file system and content addressing sounds really attractive, IPFS still remains a toy for those tech enthusiasts after 6 years. **To me IPFS is a powerful tool, yet also powerless due to its developers' ambition to overthrow the traditional structure of web**. It would be better for it to focus on a specific field, gain more active users, and then consider the next step if its ultimate goal is to compete in a monopolized market. Despite IPFS's attempts at conquering every corner of the web that involves static content, better, easier, and more stable alternatives have all already existed which makes the invention of IPFS somehow purposeless. This is getting even more difficult regarding dynamic content.

Common usages of IPFS involve static website hosting, file backup, file sharing, etc. As I stated previously, IPFS is capable at completing many tasks, but excel at none of them. The speed of IPFS is also concerning (or maybe I should put the blame on my computer rather tha IPFS) compared to other data exchanging services. We are now already at a shortage of nodes as its user base remains small.

One common misconception for IPFS that tempts some users to try it is that it is private, but in fact it is not if you are running it on open network without assist from file encryption, VPN/Tor, or establishment of a private IPFS network.

I would not argue that IPFS is completely useless, as its performance in the 2017 Catalan independence referendum has proved that **IPFS can be used to combat censorship under an authoritarian government that obtains massive amount of resources to block the country's access to the free world**. In 2017, the Spanish government has blocked every website about the referendum to restrain the spread of information. Soon the Catalan government moved the website to IPFS and despite the Spanish government's multiple attempts at blocking all public gateways by brutal force, new nodes always arise as people started to maintain their own nodes. Again, IPFS is not private, so many citizens who published information on the referendum were arrested by the Spanish government as a result.

You could argue that there are better alternatives for secure information sharing, like how Hong Kong protesters used the Telegram channel to broadcast news during the 2020 Anti-Extradition Law Amendment Bill Movement. The main, and allow me to say this, the only advantage of IPFS is decentralization. It does not matter how powerful and resourceful a government is, as long as there is internet, information can be spread. The only concern right now is how to find an accessible gateway without pinging every known gateway in a country where IPFS is "censored".

In fact, IPFS is not the only project who has introduced the idea of decentralized file sharing. An anonymous Chinese blogger famous for his/her criticism of the Chinese government and comprehensive privacy guides titled "Programthink" has used BitTorrent Sync, a decentralized file synchronization tool, to spread banned books in mainland China.

This is just a summary of how IPFS operates and my brief thought on it. I am looking forward to seeing more creative or practical usage of it, but right now I find no particular reason to use it. However, I would be really interested to see if the implementation of IPFS would be successful in China, as I have not seen any wide usage of it even among those who are able to pass through GFW. Yet in other countries that are not intensely censored, the probability of IPFS overtaking older protocols seems low, as IPFS aims to subvert the base system of the web.
