**Q1 [1 point].** In the `dlops-io/data-versioning` container setup, the following statements describe different stages of how data from the GCS bucket become available inside the container.
  Which combination of statements correctly explains the process and its implications?

1. The bucket gs://cheese-app-data-versioning is mounted via **FUSE** using gcsfuse into /mnt/gcs_data.

2. The command mount --bind /mnt/gcs_data/images /app/cheese_dataset maps the local view of remote data into the application workspace.

3. DVC interprets /app/cheese_dataset as a local directory, but the underlying reads/writes are redirected through the FUSE mount to the remote GCS bucket.

4. Because the bucket is mounted directly into /app, no data persist on the host after the container stops.

5. Secrets used by gcsfuse are embedded in the Docker image to simplify credential loading.

**Which subset of these statements is entirely correct?** </br>
       **A.** 1, 2, and 3 </br>
       **B.** 1, 3, and 4 </br>
       **C.** 2, 4, and 5 </br>
       **D.** 1, 2, 3, and 4 </br>


**Answer Q1:** **A.** 1, 2, and 3
Justification: gcsfuse is a FUSE (Filesystem in Userspace) layer that mounts a Google Cloud Storage (GCS) bucket as a local directory within the filesystem.The mount --bind command then remaps this mounted path inside the container's filesystem, enabling applications to interact with bucket data seamlessly as if it were stored locally. DVC interprets /app/cheese_dataset as a standard local directory, while all underlying read and write operations are transparently handled by gcsfuse and redirected to the remote GCS bucket.Since the bucket is only mounted virtually not directly, no actual data persist on the host machine once the container stops — the mount simply disappears. Secrets are not embeded in the docker image and its mounted as volume.

**Q2 [1 point].** Given the `dlops-io/data-versioning` repo and its workflow, which lines can be dropped from the docker run command **without breaking** the data-versioning flow?
      
```
 docker run --rm --name data-version-cli -ti
       A) --privileged
       B) --cap-add SYS_ADMIN
       C) --device /dev/fuse
       D) -v “BASE_DIR”:/app
       E) -v “SECRETS_DIR”:/secrets
       F) -v ~/.gitconfig:/etc/gitconfig
       G) -e GOOGLE_APPLICATION_CREDENTIALS=GOOGLE_APPLICATION_CREDENTIALS
       H) -e GCP_PROJECT=GCP_PROJECT
       I) -e GCP_ZONE=GCP_ZONE
       J) -e GCS_BUCKET_NAME=GCS_BUCKET_NAME
       k)  data-version-cli
```

**Answer Q2:**  **A,F,H and I** can be dropped without breaking the data-versioning flow.'A' is not need if 'B' and 'C' are used since gfuse needs only 'B' and 'C' for its implementation and hence 'A' becomes optional and it will not break the dataversioning flow if 'A' is removed. 'F' is only required to identify the user (author) when making Git commits. The data-versioning flow (DVC/GCS/FUSE) works without it, as DVC/GCS do not require the Git config. 'H' is optional as GCS operations can infer the project from service account. 'I' is also optinoal as its only relevant for zonal services and not for GCS access or data-versioning flow.



**Q3  [1 point].** In the dlops-io/data-versioning setup, where does the actual mount from the GCS bucket into the container occur? </br>
       **A.** In the Dockerfile during image build, via a RUN gcsfuse ... line </br>
       **B.** In docker-entrypoint.sh, which runs gcsfuse to mount gs://.../images to /mnt/gcs_data/images, then mount --bind to /app/cheese_dataset </br>
       **C.** In .dvc/config after dvc remote add -d ... </br>
       **D.** In the docker run command via -v “$BASE_DIR”:/app </br>

**Answer Q3:** **B** is the right answer. docker-entrypoint.sh executes when the container starts and it runs gcsfuse to mount the bucket and then it performs necessary bind mounts to make the data available to the application's expected path (/app/cheese_dataset) using the command `mount --bind /mnt/gcs_data/images /app/cheese_dataset`.



**Q4 [2 points].** If you add a new image file outside the container in the `cheese_dataset` folder, does it appear in the GCS bucket? Explain why or why not, and identify the lines responsible for this behavior.

**Answer Q4:** No, the new image file outside the container in the cheese_dataset will not appear in GCS bucket becuase the data-versioning flow depends on the FUSE mount inside the container, not the host folder.
The behavior is determined by the following lines:
in docker-shell.sh:
-v "$BASE_DIR":/app
in the docker-entrypoint.sh
1. `gcsfuse gs://$GCS_BUCKET_NAME/images /mnt/gcs_data/images`
2. `mount --bind /mnt/gcs_data/images /app/cheese_dataset`

**Q5 [2 points].** In the `docker-entrypoint.sh` there is a line `mkdir -p /app/cheese_dataset`. Is this line necessary? If yes, explain why it is included and what would happen if it were removed.

**Answer Q5:**   The mkdir -p command ensures that the directory /app/cheese_dataset exists before the mount --bind operation maps the FUSE-mounted bucket into it while running the container through docker-entrypoint.sh. If the line is missing docker-entrypoint.sh will give runtime error as it needs this directory /app/cheese_dataset to exist before it can mount bind to gcs bucket, in case if the directory doesn't alreedy exist.



**Q6 [2 points]** . In the README file we have this instruction 

**#### Update Git to track DVC** 

Run this outside the container. 

Given the current setup, do we actually need to perform this step outside the container? If not, explain why, and identify where in the code this step is already handled.

**Answer Q6:**   No, we don't need to perform this step outside the container. The docker file line number 4 `ARG DEBIAN_PACKAGES="build-essential git curl wget unzip gzip"`, ensures git is available in the container, which is the reason why the "update Git to track DVC" can be run either inside or outside the container.
