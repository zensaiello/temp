### Some Options:

-----

#### 1. Just disable PCS from starting serviced

1. Disable passive node

       pcs cluster standby node2
       
1. Disable serviced pcs resource

       pcs resource disable serviced

   On bootup, will need to confirm pcs resources started fine, `pcs status`. Once confirmed, manually start serviced, `systemctl start serviced && journalctl -fu serviced`.
   
   **Note**: Ensure you stop serviced, `systemctl stop serviced` before stoping any pcs configured resource or before any server reboot or shutdown.

-----

#### 2. Remove HA: recreate storage, restore from backup
1. Stop serviced
   1. stop resmgr: `serviced service stop Zenoss.resmgr`
   1. resource hosts: `systemctl stop serviced`
   1. master: `pcs resource disable serviced`
   1. zk ensemble hosts: `systemctl stop serviced`
1. Note VirtualIP: `pcs resource show VirtualIP`
1. Note disk layout, paying specific attention to serviced thinpool & isvcs. `lsblk`
1. Stop/Disable HA services:

        pcs resource disable serviced-group
        # Wait for pcs status to show everything as stopped
        systemctl stop pcsd; systemctl stop cotosync; systemctl stop pacemaker
        systemctl disable pcsd; systemctl disable cotosync; systemctl disable pacemaker
1. Disable drbdm
   1. move isvcs data to temporary location
      `/opt/serviced/var/isvcs/.keys` directory ?
      
          cp -r /opt/serviced/var/isvcs /root/isvcs-tmp
          umount /opt/serviced/var/isvcs
   
   1. proceed with disabling drbd
 
          drbdadm down all
          mv /etc/drbd.d/serviced-dfs.res ~/
          vi /etc/lvm/lvm.conf 
          # comment out: filter = ["r|/dev/sdd|"]
          # originally that line was a commented example of: filter = [ "a|.*/|" ] 

1. Recreate Storage:
   1. serviced thinpool: 
      **NOTE**: replace `/dev/sde` with the appropriate device noted above

          wipefs -a /dev/sde
          serviced-storage create-thin-pool serviced /dev/sde:

   1. `/opt/serviced/var/isvcs` volume
       **NOTE**: replace `/dev/sdd` with the appropriate device noted above.
       
          wipefs -a /dev/sdd
          mkfs.xfs /dev/sdd
          vi /etc/fstab  # make changes to the isvcs entry; UUID not the same, etc.

1. Re-Bind virtual IP:
    1. Temporary, not persistent through boot

           ip address add 10.60.61.62/24 dev eth0

    1. Whatever long term methoodology used by client's supporting infra team
       *Internal Ticker* ?

1. Start enable:
    1. zk ensemble hosts: `systemctl start serviced && journalctl -fu serviced`
    1. master: `systemctl start serviced && journalctl -fu serviced`
       **WAIT**: serviced to start on master, verify working well on master & ZK ensemble nodes
    1. master: `systemctl enable serviced`
    1. resource hosts: `systemctl start serviced && journalctl -fu serviced`
       **WAIT**: verify nodes are working and show as up CC web ui 
       systemctl start serviced && journalctl -fu serviced
    1. Start ResourceManager:`serviced service start Zenoss.resmgr

1. Restore backup.

---
       
#### 3. Migrate storage to nonDRBD volumes

> **Paul Fielding**: Here's what I would do to get of of DRBD storage (I've done this before when completely rebuilding someone's DRBD Storage, but same would apply to get off of it). I'd still take a backup first, just to be safe, but should be pretty safe if you're careful.

1. Follow same procedure outlined in **Option#2**, except **Step 6.** "*Recreate Storage*"
1. **Migrate Storage**:
   1. **serviced volume**
   
      - get a *temporary swing volume* or *new volume data will be migrated to* at least as big as original serviced volume - can be reclaimed after
      - add swing volume to serviced volume group
         1. Identify HA/DRBD  volume

                $ lsblk -p --output=NAME,SIZE,TYPE
                NAME                                                        SIZE TYPE
                /dev/sde                                                     90G disk
                └─/dev/drbd2                                                 90G disk
                  ├─/dev/mapper/serviced-serviced--pool_tmeta                96M lvm
                  │ └─/dev/mapper/serviced-serviced--pool                  80.9G lvm
                  │   └─/dev/mapper/docker-147:1-67-2Op2dvqhGfA6gb6El3QxV9   45G dm
                  └─/dev/mapper/serviced-serviced--pool_tdata              80.9G lvm
                    └─/dev/mapper/serviced-serviced--pool                  80.9G lvm
                      └─/dev/mapper/docker-147:1-67-2Op2dvqhGfA6gb6El3QxV9   45G dm

         1. Identify new volume
           
                $ lsblk -p --output=NAME,SIZE,TYPE
                NAME     SIZE TYPE
                /dev/sdf  95G disk
                  
         1. Create LVM physical Volume; `pvcreate /dev/sdf`
         1. Extend the serviced lvm volume group; `vgextend serviced /dev/sdf`

      - pvmigrate the extents to the swing volume. *takes time & very IO intensive*

            pvmove /dev/drbd2 /dev/sdf
      - remove drbd volume from volume group; `vgreduce serviced /dev/drbd2`
      - remove that drbd volume from drbd config: **/etc/drbd.d/serviced-dfs.res**

      **------------**
      **DONE** if only migrating data to a new volume. You can disconnect the old storage from the server
         **NOTE**: rebooting server may change the device names of the storage.
      **------------**
      - rebuild the volume without drbd - **?** - `wipefs -a /dev/sde`
      - add back into serviced volume group - **?** - `pvcreate /dev/sde`
      - pvmigrate back to original device - **?** - `pvmove /dev/sdf /dev/sde`
      - remove swing volume from volume group - **?** - `vgreduce serviced /dev/sde`
      - give back swing volume - **?** - *disconnect storage*

   1. **isvcs volume**
   A short outage will be needed:

      - stop **Zenoss.resmgr** & **serviced**
      - tar out /opt/serviced/var/isvcs
      - blow away and recreate them without drbd
        **NOTE**: replace `/dev/sdd` with the appropriate device

            wipefs -a /dev/sdd
            mkfs.xfs /dev/sdd
            vi /etc/fstab  # make changes to the isvcs entry; UUID not the same, etc.
            mount -a
      - tar back in isvcs and var/volumes
      - bring serviced back up.

   **Reference**:
  
    - Zenoss -> Resource Manager -> System Administration & Configuration
      - [Zenoss Best Practices For Post-Installation Control Center Storage Management](https://support.zenoss.com/hc/en-us/articles/212165343-Zenoss-Best-Practices-for-Post-Installation-Control-Center-Storage-Management) 
        - [KB_212165343_post_install_CC_storage_management_ver_p1.pdf](https://support.zenoss.com/hc/en-us/article_attachments/207881926/KB_212165343_post_install_CC_storage_management_ver_p1.pdf)


---
