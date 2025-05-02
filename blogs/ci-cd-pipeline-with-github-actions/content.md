## Table of Contents

1. [Introduction](#introduction)
2. [Innovative Deployment Strategies](#innovative-deployment-strategies)
    - [Dynamic Slot Management](#dynamic-slot-management)
    - [Secure and Efficient Image Transfer](#secure-and-efficient-image-transfer)
    - [Synchronizing Frontend and Backend Deployments](#synchronizing-frontend-and-backend-deployments)
    - [Automating Environment Configuration](#automating-environment-configuration)
3. [Frontend CI/CD Pipeline Implementation](#frontend-cicd-pipeline-implementation)
    - [Development Workflow](#development-workflow)
    - [Production Workflow](#production-workflow)
4. [Backend CI/CD Pipeline Implementation](#backend-cicd-pipeline-implementation)
    - [Development Workflow](#development-workflow-1)
    - [Production Workflow](#production-workflow-1)
5. [Lessons Learned and Best Practices](#lessons-learned-and-best-practices)
    - [Lessons Learned](#lessons-learned)
    - [Best Practices](#best-practices)
6. [Conclusion and Future Enhancements](#conclusion-and-future-enhancements)
    - [Conclusion](#conclusion)
    - [Future Enhancements](#future-enhancements)

## Introduction

In the fast-paced world of software development, having a reliable and efficient CI/CD pipeline is crucial for delivering high-quality applications. This blog explores my journey of setting up a comprehensive CI/CD pipeline using GitHub Actions for an Angular frontend and an Express backend.

My goal was to streamline the deployment process, reduce manual intervention, and ensure that both the frontend and backend are always in sync. Through this blog, I will delve into the specific challenges I faced, the innovative solutions I implemented, and the expertise I gained along the way. Whether you're looking to enhance your own CI/CD processes or simply curious about the intricacies of deploying complex applications, this blog offers valuable insights and practical strategies.

<p align="center">
    <img src="blogs/ci-cd-pipeline-with-github-actions/assets/ci-cd-image.png" alt="CI/CD Pipeline Image" width="300"/>
</p>

## Innovative Deployment Strategies

While setting up the CI/CD pipelines for my Angular frontend and Express backend, I encountered several challenges that required creative solutions. Here's a look at some of the innovative strategies I implemented to tackle these issues and enhance the deployment process.

### Dynamic Slot Management

**The Challenge:**
With multiple environments and frequent deployments, managing deployment slots efficiently was a real headache. I needed a way to ensure that each deployment used the correct slot without causing conflicts, allowing for smooth transitions between different versions of the application.

**My Approach:**
I came up with a dynamic slot management system that automatically picks the next available deployment slot for both the frontend and backend. This system updates slot files to keep track of the last used slots, ensuring that each deployment is isolated and doesn't interfere with others. By automating slot selection, I minimized the risk of deployment conflicts and cut down on manual work.

### Secure and Efficient Image Transfer

**The Challenge:**
Transferring Docker images securely and efficiently to the server was crucial, especially given the size of the images and the need for frequent updates. I needed a fast and secure transfer process to minimize downtime during deployments.

**My Approach:**
I used `sshpass` and `scp` for secure file transfers, along with Docker's image saving and loading features. This combination ensured that images were transferred quickly and securely, reducing deployment time and boosting the reliability of the process.

**Note:** Currently, I am not utilizing any third-party services like AWS ECR or Docker Registry to store Docker images. Instead, images are managed and transferred directly between local environments and the deployment server. This approach simplifies the setup but may limit scalability and flexibility in the long term.

### Synchronizing Frontend and Backend Deployments

**The Challenge:**
Keeping the frontend and backend in sync during deployments was essential to avoid compatibility issues and ensure a smooth user experience. Mismatches between frontend and backend versions could lead to errors and inconsistencies.

**My Approach:**
I developed a system to capture and store commit hashes for both frontend and backend deployments. This allowed me to verify that the correct versions were deployed together, reducing the risk of mismatches and ensuring consistent functionality across the application. By maintaining a record of deployed versions, I could easily track and manage deployments, ensuring that both parts of the application were always aligned.

<p align="center">
    <img src="blogs/ci-cd-pipeline-with-github-actions/assets/github-actions.jpg" alt="Github Actions" width="500"/>
</p>

### Automating Environment Configuration

**The Challenge:**
Configuring environment variables dynamically for each deployment was necessary to ensure that the application pointed to the correct services and endpoints. Manual configuration was error-prone and time-consuming.

**My Approach:**
I automated the modification of environment configuration files (`environment.ts`) during the build process. This automation ensured that each deployment had the correct URLs and settings, tailored to the specific deployment slot being used. By automating environment configuration, I eliminated the risk of human error and streamlined the deployment process.

These strategies not only addressed the specific challenges I faced but also made the entire deployment process more robust and scalable. It was a rewarding experience to see these solutions come together and significantly improve the efficiency of my CI/CD pipeline.

## Frontend CI/CD Pipeline Implementation

The frontend of my application is built using Angular, a popular framework for building dynamic web applications. Setting up a CI/CD pipeline for the Angular frontend involved creating workflows that automate the build, test, and deployment processes, ensuring that new features and updates are delivered efficiently.

### Development Workflow

The development workflow is triggered by pull requests, allowing me to test changes in a controlled environment before merging them into the main branch. Here's how the workflow is structured:

-   **Trigger:** The workflow is activated on pull requests that are opened, synchronized, or reopened, except for those from the `main` branch.

    ```yaml
    on:
        pull_request:
            types: [opened, synchronize, reopened]
    ```

-   **Environment Setup:** The workflow begins by checking out the code from the pull request and setting up Docker Buildx for building Docker images.

    ```yaml
    steps:
        - name: Checkout repository
          uses: actions/checkout@v2
          with:
              ref: ${{ github.event.pull_request.head.sha }}

        - name: Set up Docker Buildx
          uses: docker/setup-buildx-action@v1
    ```

-   **Slot Management:** A dynamic slot management system selects the next available deployment slot, ensuring that each deployment is isolated and does not interfere with others.

    ```yaml
    - name: Select Deployment Slot
      id: get_slot
      run: |
          SYNCED_SLOT=$(sshpass -p "${{ secrets.SSH_PRIVATE_KEY }}" ssh -T -o StrictHostKeyChecking=no ${{ secrets.SSH_USER }}@${{ secrets.SERVER_IP }} << 'EOF' | tail -n 1
            # Logic to determine the next slot
          EOF
          )
          echo "DEPLOY_SLOT=$(echo $SYNCED_SLOT | tr -d '\r')" >> $GITHUB_ENV
    ```

-   **Docker Image Management:** The Angular application is built into a Docker image, which is then transferred to the selected deployment slot on the server.

    ```yaml
    - name: Build & tag Docker image
      run: |
          docker build -t angular-app-dev-slot-${{ env.DEPLOY_SLOT }} .
          docker save angular-app-dev-slot-${{ env.DEPLOY_SLOT }} -o angular-app-dev-slot-${{ env.DEPLOY_SLOT }}.tar
    ```

-   **Post-Deployment Actions:** After deployment, several steps are taken to ensure the deployment is successful and visible to the team:

    -   **Update Slot Files:** The workflow updates slot files to keep track of the last used slots for both frontend and backend, ensuring that future deployments use the correct slots.

        ```yaml
        - name: Update FE/BE last slot text file
          run: |
              sshpass -p "${{ secrets.SSH_PRIVATE_KEY }}" ssh -o StrictHostKeyChecking=no ${{ secrets.SSH_USER }}@${{ secrets.SERVER_IP }} "
              set -euo pipefail
              # Define the paths to the FE and BE slot files
              BE_SLOT_FILE='/path/to/be/last_slot.txt'
              FE_SLOT_FILE='/path/to/fe/last_slot.txt'

              # Update slot files
              echo \"${{ env.DEPLOY_SLOT }}\" > \"\$BE_SLOT_FILE\"
              echo \"${{ env.DEPLOY_SLOT }}\" > \"\$FE_SLOT_FILE\"
              "
        ```

    -   **Redeploy Last Deployed Backend Image:** After deploying the Angular app, the workflow checks if the backend image needs to be redeployed. It reads the last deployed backend's commit hash from a remote file and uses this commit to pull the latest code. The backend Docker image is then rebuilt and pushed to the remote server. This ensures that both the frontend and backend remain in sync and are deployed together, minimizing the risk of version mismatches and ensuring consistent functionality across the application.

        ```yaml
        # Determine if BE Image Copy is Required
        - name: Determine if BE Image Copy is Required
          id: setup_be_copy
          env:
              DEPLOY_SLOT: ${{ env.DEPLOY_SLOT }}
              LATEST_BE_SLOT: ${{ env.LATEST_BE_SLOT }}
          run: |
              if [[ "$DEPLOY_SLOT" == "$LATEST_BE_SLOT" ]]; then
                echo "✅ BE slot is the same as current FE slot. Skipping copy."
                echo "skip=true" >> $GITHUB_OUTPUT
              else
                echo "ℹ️ BE image needs to be copied to new slot."
                echo "skip=false" >> $GITHUB_OUTPUT
              fi

        # Get the commit hash for the latest deployed BE
        - name: Get the commit hash for the latest deployed BE_COMMIT
          run: |
              SYNCED_BE_COMMIT=$(sshpass -p "${{ secrets.SSH_PRIVATE_KEY }}" ssh -T -o StrictHostKeyChecking=no ${{ secrets.SSH_USER }}@${{ secrets.SERVER_IP }} bash << 'EOF'
                set -e
                SLOT_DETAILS_FILE="/home/${{ secrets.SSH_USER }}/Desktop/dev-deployment/docker-images/slot0${{ env.LATEST_BE_SLOT }}-details.txt"

                if [[ -f "$SLOT_DETAILS_FILE" ]]; then
                  grep "be-" "$SLOT_DETAILS_FILE" | awk '{print $2}'
                fi
              EOF
              )

              echo "Fetched BE Commit: $SYNCED_BE_COMMIT"
              echo "BE_COMMIT_HASH=$SYNCED_BE_COMMIT" >> $GITHUB_ENV

        # Build docker image for BE
        - name: Build docker image for BE
          if: steps.setup_be_copy.outputs.skip == 'false'
          run: |
              echo "Using BE Commit: $BE_COMMIT_HASH"

              # Clone the repository
              git clone https://${{ secrets.ORG_NAME }}:${{ secrets.PAT_TOKEN }}@github.com/binapani-edu/academy-backend-api.git be-src
              cd be-src
              git checkout $BE_COMMIT_HASH

              # Build the Docker image
              docker build -t express-app-dev-slot-${{ env.DEPLOY_SLOT }} .

              # Save the Docker image
              docker save express-app-dev-slot-${{ env.DEPLOY_SLOT }} -o express-app-dev-slot-${{ env.DEPLOY_SLOT }}.tar

              # Move it to the root of the runner
              mv express-app-dev-slot-${{ env.DEPLOY_SLOT }}.tar ../

        # Transfer Docker image to the selected slot
        - name: Transfer Docker image to the selected slot
          if: steps.setup_be_copy.outputs.skip == 'false'
          uses: appleboy/scp-action@v0.1.2
          with:
              host: ${{ secrets.SERVER_IP }}
              username: ${{ secrets.SSH_USER }}
              password: ${{ secrets.SSH_PRIVATE_KEY }}
              source: "./express-app-dev-slot-${{ env.DEPLOY_SLOT }}.tar"
              target: "/home/${{ secrets.SSH_USER }}/Desktop/dev-deployment/docker-images/be/slot-${{ env.DEPLOY_SLOT }}"

        # Restart only the updated Backend container
        - name: Restart Backend Service and Update Last Slot File
          if: steps.setup_be_copy.outputs.skip == 'false'
          run: |
              sshpass -p "${{ secrets.SSH_PRIVATE_KEY }}" ssh -o StrictHostKeyChecking=no ${{ secrets.SSH_USER }}@${{ secrets.SERVER_IP }} "
                echo '${{ secrets.SSH_PRIVATE_KEY }}' | sudo -S docker load -i /home/${{ secrets.SSH_USER }}/Desktop/dev-deployment/docker-images/be/slot-${{ env.DEPLOY_SLOT }}/express-app-dev-slot-${{ env.DEPLOY_SLOT }}.tar

                cd /home/${{ secrets.SSH_USER }}/Desktop/dev-deployment
                echo '${{ secrets.SSH_PRIVATE_KEY }}' | sudo -S docker-compose up -d express-app-slot-${{ env.DEPLOY_SLOT }}

                echo ${{ env.DEPLOY_SLOT }} > /home/${{ secrets.SSH_USER }}/Desktop/dev-deployment/docker-images/be/last_slot.txt
              "
        ```

        This snippet provides a clear view of the steps involved in determining if the backend image needs to be redeployed, fetching the commit hash, building the Docker image, and deploying it to the server.

    -   **Post Deployment Comment:** A comment is posted on the pull request with deployment details, providing visibility into the deployment process and ensuring that all team members are informed of the latest changes.

        ```yaml
        - name: Post Deployment Comment on PR
          uses: actions/github-script@v6
          with:
              github-token: ${{ secrets.GITHUB_TOKEN }}
              script: |
                  const slot = process.env.DEPLOY_SLOT;
                  const devUrl = `https://dev-url-for-slot-${slot}.example.com`;
                  const commentBody = `
                  🚀 **Deployment is Successful!**
                  - **Frontend Slot:** ${slot}
                  - **Dev URL:** [${devUrl}](${devUrl})
                  ✅ The Angular app has been successfully deployed.
                  `;

                  const issue_number = context.payload.pull_request.number;
                  const owner = context.repo.owner;
                  const repo = context.repo.repo;

                  await github.request('POST /repos/{owner}/{repo}/issues/{issue_number}/comments', {
                    owner,
                    repo,
                    issue_number,
                    body: commentBody,
                  });
        ```

<p align="center">
    <img src="blogs/ci-cd-pipeline-with-github-actions/assets/bulky-cd.png" alt="Bulky Continuous Deployment" width="300"/>
    <br/>
    <em>"When your CI/CD pipeline feels as bulky as this CD, but still gets the job done!"</em>
</p>

### Production Workflow

The production workflow is designed to deploy stable releases to the live environment, ensuring that users always have access to the latest features and improvements. Here's an overview of the production workflow:

-   **Trigger:** The workflow is triggered by pushes to the `main` branch, indicating that the code is ready for production.

    ```yaml
    on:
        push:
            branches:
                - main
    ```

-   **Environment Setup:** Similar to the development workflow, the code is checked out, and Docker Buildx is set up.

    ```yaml
    steps:
        - name: Checkout repository
          uses: actions/checkout@v2

        - name: Set up Docker Buildx
          uses: docker/setup-buildx-action@v1
    ```

-   **Docker Image Management:** The Angular application is built into a Docker image and transferred to the server.

    ```yaml
    - name: Build Docker image for Angular app
      run: |
          docker build -t angular-app:latest .
          docker save angular-app:latest -o angular-app.tar
    ```

-   **Deployment:** The Docker image is loaded on the server, and the `angular-app` service is updated using Docker Compose, ensuring a seamless transition to the new version.

    ```yaml
    - name: Load and deploy Docker image on the server
      run: |
          sshpass -p "${{ secrets.SSH_PRIVATE_KEY }}" ssh -o StrictHostKeyChecking=no ${{ secrets.SSH_USER }}@${{ secrets.SERVER_IP }} "
            echo '${{ secrets.SSH_PRIVATE_KEY }}' | sudo -S docker load -i /path/to/angular-app.tar && \
            cd /path/to/deployment && \
            echo '${{ secrets.SSH_PRIVATE_KEY }}' | sudo -S docker-compose up -d angular-app
          "
    ```

By automating these processes with GitHub Actions, I have achieved a streamlined and efficient CI/CD pipeline for the Angular frontend, reducing manual intervention and accelerating the delivery of new features.

## Backend CI/CD Pipeline Implementation

The backend of my application is powered by an Express server, which handles API requests and business logic. Setting up a CI/CD pipeline for the Express backend involved creating workflows that automate the build, test, and deployment processes, ensuring that the backend is always up-to-date and in sync with the frontend.

### Development Workflow

The development workflow for the backend is triggered by pull requests, allowing me to test changes in a controlled environment before merging them into the main branch. Here's how the workflow is structured:

-   **Trigger:** The workflow is activated on pull requests that are opened, synchronized, or reopened, except for those from the `main` branch.

    ```yaml
    on:
        pull_request:
            types: [opened, synchronize, reopened]
    ```

-   **Environment Setup:** The workflow begins by checking out the code from the pull request and setting up Docker Buildx for building Docker images.

    ```yaml
    steps:
        - name: Checkout repository
          uses: actions/checkout@v2
          with:
              ref: ${{ github.event.pull_request.head.sha }}

        - name: Set up Docker Buildx
          uses: docker/setup-buildx-action@v1
    ```

-   **Slot Management:** A dynamic slot management system selects the next available deployment slot for the backend, ensuring that each deployment is isolated and does not interfere with others.

    ```yaml
    - name: Select Deployment Slot
      id: get_slot
      run: |
          SYNCED_SLOT=$(sshpass -p "${{ secrets.SSH_PRIVATE_KEY }}" ssh -T -o StrictHostKeyChecking=no ${{ secrets.SSH_USER }}@${{ secrets.SERVER_IP }} << 'EOF' | tail -n 1
            # Logic to determine the next slot
          EOF
          )
          echo "DEPLOY_SLOT=$(echo $SYNCED_SLOT | tr -d '\r')" >> $GITHUB_ENV
    ```

-   **Docker Image Management:** The Express application is built into a Docker image, which is then transferred to the selected deployment slot on the server.

    ```yaml
    - name: Build & tag Docker image
      run: |
          docker build -t express-app-dev-slot-${{ env.DEPLOY_SLOT }} .
          docker save express-app-dev-slot-${{ env.DEPLOY_SLOT }} -o express-app-dev-slot-${{ env.DEPLOY_SLOT }}.tar
    ```

-   **Post-Deployment Actions:** After deployment, several steps are taken to ensure the deployment is successful and visible to the team:

    -   **Update Slot Files:** The workflow updates slot files to keep track of the last used slots for both frontend and backend, ensuring that future deployments use the correct slots.

        ```yaml
        - name: Update FE/BE last slot text file
          run: |
              sshpass -p "${{ secrets.SSH_PRIVATE_KEY }}" ssh -o StrictHostKeyChecking=no ${{ secrets.SSH_USER }}@${{ secrets.SERVER_IP }} "
              set -euo pipefail
              # Define the paths to the FE and BE slot files
              BE_SLOT_FILE='/path/to/be/last_slot.txt'
              FE_SLOT_FILE='/path/to/fe/last_slot.txt'

              # Update slot files
              echo \"${{ env.DEPLOY_SLOT }}\" > \"\$BE_SLOT_FILE\"
              echo \"${{ env.DEPLOY_SLOT }}\" > \"\$FE_SLOT_FILE\"
              "
        ```

    -   **Redeploy Last Deployed Frontend Image:** After deploying the Express app, the workflow checks if the frontend image needs to be redeployed. It reads the last deployed frontend's commit hash from a remote file and uses this commit to pull the latest code. The frontend Docker image is then rebuilt and pushed to the remote server. This ensures that both the frontend and backend remain in sync and are deployed together, minimizing the risk of version mismatches and ensuring consistent functionality across the application.

        ```yaml
        # Determine if FE Image Copy is Required
        - name: Determine if FE Image Copy is Required
          id: setup_fe_copy
          env:
              DEPLOY_SLOT: ${{ env.DEPLOY_SLOT }}
              LATEST_FE_SLOT: ${{ env.LATEST_FE_SLOT }}
          run: |
              if [[ "$DEPLOY_SLOT" == "$LATEST_FE_SLOT" ]]; then
                echo "✅ FE slot is the same as current BE slot. Skipping copy."
                echo "skip=true" >> $GITHUB_OUTPUT
              else
                echo "ℹ️ FE image needs to be copied to new slot."
                echo "skip=false" >> $GITHUB_OUTPUT
              fi

        # Get the commit hash for the latest deployed FE
        - name: Get the commit hash for the latest deployed FE_COMMIT
          run: |
              SYNCED_FE_COMMIT=$(sshpass -p "${{ secrets.SSH_PRIVATE_KEY }}" ssh -T -o StrictHostKeyChecking=no ${{ secrets.SSH_USER }}@${{ secrets.SERVER_IP }} bash << 'EOF'
                set -e
                SLOT_DETAILS_FILE="/home/${{ secrets.SSH_USER }}/Desktop/dev-deployment/docker-images/slot0${{ env.LATEST_FE_SLOT }}-details.txt"

                if [[ -f "$SLOT_DETAILS_FILE" ]]; then
                  grep "fe-" "$SLOT_DETAILS_FILE" | awk '{print $2}'
                fi
              EOF
              )

              echo "Fetched FE Commit: $SYNCED_FE_COMMIT"
              echo "FE_COMMIT_HASH=$SYNCED_FE_COMMIT" >> $GITHUB_ENV

        # Build docker image for FE
        - name: Build docker image for FE
          if: steps.setup_fe_copy.outputs.skip == 'false'
          run: |
              echo "Using FE Commit: $FE_COMMIT_HASH"

              # Clone the repository
              git clone https://${{ secrets.ORG_NAME }}:${{ secrets.PAT_TOKEN }}@github.com/binapani-edu/academy-frontend.git fe-src
              cd fe-src
              git checkout $FE_COMMIT_HASH

              # Build the Docker image
              docker build -t angular-app-dev-slot-${{ env.DEPLOY_SLOT }} .

              # Save the Docker image
              docker save angular-app-dev-slot-${{ env.DEPLOY_SLOT }} -o angular-app-dev-slot-${{ env.DEPLOY_SLOT }}.tar

              # Move it to the root of the runner
              mv angular-app-dev-slot-${{ env.DEPLOY_SLOT }}.tar ../

        # Transfer Docker image to the selected slot
        - name: Transfer Docker image to the selected slot
          if: steps.setup_fe_copy.outputs.skip == 'false'
          uses: appleboy/scp-action@v0.1.2
          with:
              host: ${{ secrets.SERVER_IP }}
              username: ${{ secrets.SSH_USER }}
              password: ${{ secrets.SSH_PRIVATE_KEY }}
              source: "./angular-app-dev-slot-${{ env.DEPLOY_SLOT }}.tar"
              target: "/home/${{ secrets.SSH_USER }}/Desktop/dev-deployment/docker-images/fe/slot-${{ env.DEPLOY_SLOT }}"

        # Restart only the updated Frontend container
        - name: Restart Frontend Service and Update Last Slot File
          if: steps.setup_fe_copy.outputs.skip == 'false'
          run: |
              sshpass -p "${{ secrets.SSH_PRIVATE_KEY }}" ssh -o StrictHostKeyChecking=no ${{ secrets.SSH_USER }}@${{ secrets.SERVER_IP }} "
                echo '${{ secrets.SSH_PRIVATE_KEY }}' | sudo -S docker load -i /home/${{ secrets.SSH_USER }}/Desktop/dev-deployment/docker-images/fe/slot-${{ env.DEPLOY_SLOT }}/angular-app-dev-slot-${{ env.DEPLOY_SLOT }}.tar

                cd /home/${{ secrets.SSH_USER }}/Desktop/dev-deployment
                echo '${{ secrets.SSH_PRIVATE_KEY }}' | sudo -S docker-compose up -d angular-app-slot-${{ env.DEPLOY_SLOT }}

                echo ${{ env.DEPLOY_SLOT }} > /home/${{ secrets.SSH_USER }}/Desktop/dev-deployment/docker-images/fe/last_slot.txt
              "
        ```

        This snippet provides a clear view of the steps involved in determining if the frontend image needs to be redeployed, fetching the commit hash, building the Docker image, and deploying it to the server.

    -   **Post Deployment Comment:** A comment is posted on the pull request with deployment details, providing visibility into the deployment process and ensuring that all team members are informed of the latest changes.

        ```yaml
        - name: Post Deployment Comment on PR
          uses: actions/github-script@v6
          with:
              github-token: ${{ secrets.GITHUB_TOKEN }}
              script: |
                  const slot = process.env.DEPLOY_SLOT;
                  const devUrl = `https://dev-url-for-slot-${slot}.example.com/api`;
                  const commentBody = `
                  🚀 **Deployment is Successful!**
                  - **Backend Slot:** ${slot}
                  - **API URL:** [${devUrl}](${devUrl})
                  ✅ The Express app has been successfully deployed.
                  `;

                  const issue_number = context.payload.pull_request.number;
                  const owner = context.repo.owner;
                  const repo = context.repo.repo;

                  await github.request('POST /repos/{owner}/{repo}/issues/{issue_number}/comments', {
                    owner,
                    repo,
                    issue_number,
                    body: commentBody,
                  });
        ```

### Production Workflow

The production workflow is designed to deploy stable releases to the live environment, ensuring that users always have access to the latest features and improvements. Here's an overview of the production workflow:

-   **Trigger:** The workflow is triggered by pushes to the `main` branch, indicating that the code is ready for production.

    ```yaml
    on:
        push:
            branches:
                - main
    ```

-   **Environment Setup:** Similar to the development workflow, the code is checked out, and Docker Buildx is set up.

    ```yaml
    steps:
        - name: Checkout repository
          uses: actions/checkout@v2

        - name: Set up Docker Buildx
          uses: docker/setup-buildx-action@v1
    ```

-   **Docker Image Management:** The Express application is built into a Docker image and transferred to the server.

    ```yaml
    - name: Build Docker image for Express app
      run: |
          docker build -t express-app:latest .
          docker save express-app:latest -o express-app.tar
    ```

-   **Deployment:** The Docker image is loaded on the server, and the `express-app` service is updated using Docker Compose, ensuring a seamless transition to the new version.

    ```yaml
    - name: Load and deploy Docker image on the server
      run: |
          sshpass -p "${{ secrets.SSH_PRIVATE_KEY }}" ssh -o StrictHostKeyChecking=no ${{ secrets.SSH_USER }}@${{ secrets.SERVER_IP }} "
            echo '${{ secrets.SSH_PRIVATE_KEY }}' | sudo -S docker load -i /path/to/express-app.tar && \
            cd /path/to/deployment && \
            echo '${{ secrets.SSH_PRIVATE_KEY }}' | sudo -S docker-compose up -d express-app
          "
    ```

By automating these processes with GitHub Actions, I have achieved a streamlined and efficient CI/CD pipeline for the Express backend, reducing manual intervention and ensuring that the backend is always in sync with the frontend.

## Lessons Learned and Best Practices

Embarking on the journey to set up a CI/CD pipeline for my Angular frontend and Express backend taught me a lot. Here are some of the key lessons I learned along the way, along with best practices that emerged from the experience.

### Lessons Learned

1. **Automation is Key:**
   Automating repetitive tasks not only saves time but also reduces the risk of human error. By automating the build, test, and deployment processes, I was able to focus more on developing features and less on manual tasks.

2. **Keep It Simple:**
   It's easy to get carried away with complex solutions, but simplicity often leads to more maintainable and understandable workflows. Starting with simple workflows and gradually adding complexity as needed proved to be an effective strategy.

3. **Monitor and Iterate:**
   Continuous monitoring of the CI/CD pipeline is crucial. It helped me identify bottlenecks and areas for improvement. Regularly iterating on the workflows ensured that they remained efficient and aligned with the project's needs.

4. **Communication is Crucial:**
   Keeping the team informed about deployments and changes is vital. Automated comments on pull requests and clear documentation helped maintain transparency and collaboration.

### Best Practices

1. **Use Secrets for Sensitive Information:**
   Always store sensitive information like API keys and credentials in GitHub Secrets. This ensures that they are not exposed in the codebase and are securely managed.

2. **Version Control for Configuration:**
   Keep configuration files under version control. This allows for easy tracking of changes and ensures that configurations are consistent across environments.

3. **Test Locally Before Deploying:**
   Testing changes locally before deploying them to the CI/CD pipeline can catch issues early and save time. It's a simple step that can prevent larger problems down the line.

4. **Document Your Workflows:**
   Clear documentation of the CI/CD workflows helps onboard new team members and provides a reference for troubleshooting. It's an investment that pays off in the long run.

5. **Regularly Review and Optimize:**
   The needs of a project can change over time, so it's important to regularly review and optimize the CI/CD workflows. This ensures that they continue to meet the project's requirements and remain efficient.

By applying these lessons and best practices, I was able to create a CI/CD pipeline that is not only efficient but also adaptable to future needs. The experience has been invaluable, and I look forward to applying these insights to future projects.

<p align="center">
    <img src="blogs/ci-cd-pipeline-with-github-actions/assets/funny-angular-express-docker-union.png" alt="Funny Angular Express Docker Union" width="300"/>
</p>

## Conclusion and Future Enhancements

Setting up a CI/CD pipeline for my Angular frontend and Express backend has been a transformative journey. It has not only streamlined the deployment process but also significantly enhanced the overall efficiency and reliability of my development workflow. By automating repetitive tasks and ensuring that both the frontend and backend are always in sync, I've been able to focus more on building features and less on manual deployment tasks.

### Conclusion

Reflecting on this journey, I've come to appreciate the power of automation, simplicity, and continuous improvement. The strategies and best practices I've adopted have resulted in a robust deployment process that supports rapid development cycles. This experience has equipped me with valuable insights and skills that I can apply to future projects.

However, it's important to acknowledge the current limitations. At present, the deployment is taking place on bare metal servers. While the applications are containerized using Docker, the infrastructure is not inherently scalable. If the system becomes overwhelmed, it lacks the flexibility to scale up easily. This is a significant consideration as the application grows and user demand increases.

### Future Enhancements

To address these limitations and prepare for future growth, I'm considering a strategic shift to a cloud-based infrastructure. Transitioning to a third-party cloud provider like AWS would enable greater scalability and flexibility. Here's how I envision this transformation:

1. **Embracing Kubernetes with AWS EKS:**
   By leveraging AWS Elastic Kubernetes Service (EKS), I can orchestrate containers and manage deployments across multiple environments with ease. This would provide the scalability needed to handle increased demand seamlessly.

2. **Efficient Image Management with AWS ECR:**
   Currently, I am not using any third-party services like AWS ECR or Docker Registry to store Docker images. Transitioning to AWS ECR would allow me to securely store, manage, and deploy Docker container images, streamlining the deployment process and enhancing scalability.

3. **Scalable Computing with AWS EC2:**
   AWS Elastic Compute Cloud (EC2) offers scalable computing capacity, allowing the infrastructure to grow with demand. This flexibility is crucial for maintaining performance as the application scales.

4. **Infrastructure as Code with Terraform:**
   Automating the provisioning and management of cloud resources with Terraform ensures consistency and repeatability. This approach simplifies infrastructure management and reduces the risk of human error.

5. **Enhanced CI/CD with Jenkins and ArgoCD:**
   Integrating Jenkins and ArgoCD into the CI/CD pipeline would further automate and streamline the build, test, and deployment processes, enhancing efficiency and reliability.

6. **Code Quality and Security with SonarQube:**
   Incorporating SonarQube into the pipeline would enable code quality checks and security scanning, ensuring that the application remains robust and secure.

By focusing on these future enhancements, I aim to create a CI/CD pipeline that not only addresses current scalability limitations but also opens up new possibilities for innovation and growth. Transitioning to a cloud-based infrastructure will provide the flexibility and scalability needed to support the application's evolution, ensuring it remains a cornerstone of efficient software delivery.
