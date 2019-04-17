FROM busybox

ADD ./smb /bin/smb
ADD ./install.sh /bin/install_smb_flexvol.sh
ADD ./jq /bin/jq

ENTRYPOINT ["/bin/install_smb_flexvol.sh"]
