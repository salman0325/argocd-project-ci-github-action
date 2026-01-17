```yaml
  update_cd_repo_image_tag:
    runs-on: ubuntu-latest
    needs: build_backend_docker_image_and_push  # aapke image build job ke baad chale
    steps:
      - name: Checkout CD repo
        uses: actions/checkout@v4
        with:
          repository: your-github-username/cd-repo
          token: ${{ secrets.GITHUB_TOKEN }}   # ya personal access token agar private repo hai
          path: cd-repo

      - name: Update frontend image tag in CD repo manifest
        run: |
          sed -i "s|image: salmandevops0325/argocd-k8s-frontend:.*|image: salmandevops0325/argocd-k8s-frontend:${{ env.IMAGE_TAG }}|" cd-repo/frontend/deployment.yaml

      - name: Update backend image tag in CD repo manifest
        run: |
          sed -i "s|image: salmandevops0325/argocd-k8s-backend:.*|image: salmandevops0325/argocd-k8s-backend:${{ env.IMAGE_TAG }}|" cd-repo/backend/deployment.yaml

      - name: Commit and push updated manifests
        run: |
          cd cd-repo
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add frontend/deployment.yaml backend/deployment.yaml
          git commit -m "Update frontend and backend image tags to ${{ env.IMAGE_TAG }}"
          git push
```
