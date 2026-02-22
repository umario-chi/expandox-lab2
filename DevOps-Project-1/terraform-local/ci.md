name: CI - Build, Scan, Push, Update Manifests

# Trigger only on app code or manifest changes
on:
    push:
        branches:
            - "**"
        paths:
            - "mini-finance-app/src/**"
            - "mini-finance-app/Dockerfile"
            - "mini-finance-app/requirements.txt"
            - "k8s/**"
        paths-ignore:
            - "mini-finance-app/*.md"
            - "mini-finance-app/docs/**"

permissions:
    contents: write

jobs:
    build_and_deploy:
        runs-on: ubuntu-latest

        steps:
            # 1️⃣ Checkout code
            - uses: actions/checkout@v3
              with:
                  fetch-depth: 0

            # 2️⃣ Setup Docker Buildx
            - uses: docker/setup-buildx-action@v2

            # 3️⃣ Login Docker (main/develop only)
            - name: Login to Docker Hub
              if: github.ref == 'refs/heads/main' || github.ref == 'refs/heads/develop'
              uses: docker/login-action@v2
              with:
                  username: ${{ secrets.DOCKER_USERNAME }}
                  password: ${{ secrets.DOCKERHUB_TOKEN }}

            # 4️⃣ Build Docker Image
            - name: Build Docker Image
              id: docker_build
              run: |
                  IMAGE_TAG=lakunzy/mini-finance-app:${{ github.sha }}
                  docker build -t $IMAGE_TAG ./mini-finance-app
                  echo "IMAGE_TAG=$IMAGE_TAG" >> $GITHUB_ENV

            # 5️⃣ Scan with Trivy
            - name: Scan Docker Image
              uses: aquasecurity/trivy-action@v2
              with:
                  image-ref: ${{ env.IMAGE_TAG }}
                  format: table
                  exit-code: 1
                  severity: CRITICAL,HIGH

            # 6️⃣ Push Docker Image
            - name: Push Docker Image
              if: github.ref == 'refs/heads/main' || github.ref == 'refs/heads/develop'
              run: |
                  docker push ${{ env.IMAGE_TAG }}
                  docker tag ${{ env.IMAGE_TAG }} lakunzy/mini-finance-app:latest
                  docker push lakunzy/mini-finance-app:latest

            # 7️⃣ Update Dev Manifest
            - name: Update Dev Manifest
              if: github.ref == 'refs/heads/develop'
              env:
                  GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
              run: |
                  git config --global user.name 'github-actions[bot]'
                  git config --global user.email 'github-actions[bot]@users.noreply.github.com'
                  sed -i "s|image:.*|image: ${{ env.IMAGE_TAG }}|g" k8s/dev/manifests.yaml
                  git add k8s/dev/manifests.yaml
                  git commit -m "ci: update dev image tag to ${{ env.IMAGE_TAG }}" || echo "No changes to commit"
                  git push https://${{ github.actor }}:${GITHUB_TOKEN}@github.com/${{ github.repository }}.git HEAD:develop

            # 8️⃣ Update Prod Manifest
            - name: Update Prod Manifest
              if: github.ref == 'refs/heads/main'
              environment: production
              env:
                  GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
              run: |
                  git config --global user.name 'github-actions[bot]'
                  git config --global user.email 'github-actions[bot]@users.noreply.github.com'
                  sed -i "s|image:.*|image: ${{ env.IMAGE_TAG }}|g" k8s/production/manifests.yaml
                  git add k8s/production/manifests.yaml
                  git commit -m "ci: update prod image tag to ${{ env.IMAGE_TAG }}" || echo "No changes to commit"
                  git push https://${{ github.actor }}:${GITHUB_TOKEN}@github.com/${{ github.repository }}.git HEAD:main
