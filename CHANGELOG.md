## 202x.x.x (Unreleased)

### General
- 

### Client
- 

### Server
- 


on:
  workflow_dispatch:

env:
  GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
  CHANGES_TEMPLATE: "## 2024.1.1\n\n### General\n- \n\n### Client\n- \n\n### Server\n- \n"

name: "Release Manager"

jobs:
  get-prs:
    runs-on: ubuntu-latest
    outputs:
      pr_number: ${{ steps.get_prs.outputs.value }}
    steps:
      - uses: actions/checkout@v4
      # headがrelease/かつopenのPRを1つ取得
      - name: Get PRs
        run: "gh pr list --state open --head release/ --json number --jq '.[] | .number'"
        id: get_prs
  create-release:
    permissions:
      contents: write
      pull-requests: write
    needs: get-prs
    runs-on: ubuntu-latest
    if: ${{ needs.get-prs.outputs.pr_number == '' }}
    # 日付ベースでリリースを作成
    name: "Create Release"
    steps:
      - uses: actions/checkout@v4
      # jqでpackage.jsonから現在のバージョンを取得
      - name: Get current version
        run: echo "current_version=$(jq -r '.version' package.json)" >> $GITHUB_OUTPUT
        id: get_current_version
      # バージョンをインクリメント
      - name: Increment version
        uses: actions/script@v7
        env:
          CURRENT_VERSION: ${{ steps.get_current_version.outputs.current_version }}
        with:
          script: |
            const now = new Date();
            const year = now.toLocaleDateString('en-US', { year: 'numeric', timeZone: 'Asia/Tokyo' });
            const month = now.toLocaleDateString('en-US', { month: 'numeric', timeZone: 'Asia/Tokyo' });
            const [major, minor, patch] = process.env.CURRENT_VERSION.split('.');
            if (year !== major || month !== minor) {
              return `${year}.${month}.0`;
            } else {
              return `${major}.${minor}.${Number(patch) + 1}`;
            }
          result-encoding: string
        id: increment_version
      # バージョンをpackage.jsonに書き込み
      - name: Write version
        run: |
          jq '.version = "${{ steps.increment_version.outputs.result }}-beta.0"' package.json > package.json.tmp && mv package.json.tmp package.json
          jq '.version = "${{ steps.increment_version.outputs.result }}-beta.0"' packages/misskey-js/package.json > packages/misskey-js/package.json.tmp && mv packages/misskey-js/package.json.tmp packages/misskey-js/package.json
      # CHANGELOG.mdのUnreleasedの内容を後で使うために取得
      - name: Get changelog
        run: |
          sed -n '/## 2024.1.1/,/^## /p' CHANGELOG.md | sed -e 1d -e '$d'
        id: changelog
      # CHANGELOG.mdのバージョンの書き換え
      - name: Modify CHANGELOG.md
        run: |
          sed -i 's/## 2024.1.1/## ${{ steps.increment_version.outputs.result }}/' CHANGELOG.md
          sed -i "1i $(echo "${CHANGES_TEMPLATE}" | sed -r 's/$/\\n/' | while IFS= read -r line; do echo -n "$line"; done)" CHANGELOG.md
      # バージョンをコミット
      - name: Commit version
        run: |
          git config --local user.email "LuckyBeast@users.noreply.github.com"
          git config --local user.name "LuckyBeast"
          git commit -am "Bump version to ${{ steps.increment_version.outputs.result }}-beta.0"
      # バージョンをタグ付け
      - name: Tag version
        run: |
          git tag -a v${{ steps.increment_version.outputs.result }}-beta.0 -m "Bump version to ${{ steps.increment_version.outputs.result }}-beta.0"
      # バージョンをpush
      - name: Push version
        run: "git push origin HEAD:release/${{ steps.increment_version.outputs.result }}-beta.0 --tags"
      # リリースを作成
      - name: Create release
        run: |
          gh release create ${{ steps.increment_version.outputs.result }}-beta.0 --prerelease --title "${{ steps.increment_version.outputs.result }}-beta.0" --notes "${{ steps.changelog.outputs.stdout }}"
