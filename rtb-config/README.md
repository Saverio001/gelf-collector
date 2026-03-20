# rtb-config 

A simple script that pushes and reads configurations from RtBrick elements using RESTCONF.

---
*Warning: strictly for home-lab/self-learning use - nothing is authenticated!*
---

`rtb-config` takes as arguments:

- A *file*, via the `-i` or `--inventory` parameter; it defaults to inventory.yml in the current directory
- A mandatory argument (either `push` or `save`)

The script reads the inventory, then either saves the configuration from all nodes on disk, or loads the configuration from disk into the elements. 

The inventory files contains the CTRLD base URL, and a list of nodes and filenames, using this structure:



    defaults:
      base_url: http://ctrld-host.net17.link:19091
    nodes:
      - name: R1
        config: R1.json
      - name: R2
        config: R2.json
      - name: R3
        config: R1.json
      - name: R4
        config: R2.json    


I am running CTRLD at the URL http://ctrld-host.net17.link:19091, please replace it according to your configuration (e.g. replace host and port)