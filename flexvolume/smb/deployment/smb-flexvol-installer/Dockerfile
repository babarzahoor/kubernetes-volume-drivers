FROM busybox

ADD ./smb /bin/smb
ADD ./install.sh /bin/install_smb_flexvol.sh
ADD ./jq /bin/jq
RUN chmod 777 /bin/jq

ENTRYPOINT ["/bin/install_smb_flexvol.sh"]
