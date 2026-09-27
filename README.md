### Repository for provisioning of local KVM machines using packer image and ansible

VMs are configred using qemu agent

Can be also used with cloud image by adding image_url to variables:<br>
image_url: "https://cloud.debian.org/images/cloud/trixie/20260722-2547/debian-13-nocloud-amd64-20260722-2547.qcow2"

And replacing ansible block "Create standalone VM disk from local template" with:

    - name: Download the Ubuntu Cloud Image
      ansible.builtin.get_url:
        url: "{{ image_url }}"
        dest: "{{ disk_path }}"
        mode: "0644"
      register: download_image
      until: download_image is success
      retries: 3
      delay: 5
