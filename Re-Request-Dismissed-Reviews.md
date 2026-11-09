Re-requesting Dismissed Reviews on GitHub
============================================

When "Dismiss stale pull request approvals when new commits are pushed" dismisses an approval, GitHub doesn't re-request the reviewer, or the team they reviewed on behalf of. This workflow re-requests both. More info [here](https://joshua.hu/automatically-re-request-dismissed-reviews-teams-github).

`GITHUB_TOKEN` can't see or request teams, so this needs a GitHub App with the _Pull requests: Read and write_ and _Members: Read-only_ permissions, installed on the repository. Put its App ID in the `GH_APP_ID` variable and its private key in the `GH_APP_PRIVATE_KEY` secret.

```yaml
name: Re-request dismissed reviews

# Like pull_request, this event builds the PR merge ref, so it doesn't fire
# on PRs with merge conflicts.
on:
  pull_request_review:
    types: [dismissed]

permissions: {}

jobs:
  re-request:
    # Skipped for fork PRs and Dependabot (no secrets on this event) and bot
    # reviewers (can't be requested).
    if: >-
      github.event.pull_request.state == 'open' &&
      github.event.pull_request.head.repo.full_name == github.repository &&
      github.actor != 'dependabot[bot]' &&
      github.event.review.user.type == 'User'
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      # github.token can neither see teams nor request them, so team re-requests need an app token.
      - name: Mint scoped token
        id: app-token
        uses: actions/create-github-app-token@bcd2ba49218906704ab6c1aa796996da409d3eb1 # v3.2.0
        with:
          app-id: ${{ vars.GH_APP_ID }}
          private-key: ${{ secrets.GH_APP_PRIVATE_KEY }}
          permission-pull-requests: write
          permission-members: read

      # A team member's review satisfies the team's request and removes it, so
      # re-request each team the review was on behalf of that no other approval
      # still covers, and the reviewer unless they are the sender (they
      # dismissed it, or their push did).
      - name: Re-request review
        env:
          GH_TOKEN: ${{ steps.app-token.outputs.token }}
          PR_NUMBER: ${{ github.event.pull_request.number }}
          REVIEW_ID: ${{ github.event.review.node_id }}
          REVIEWER: ${{ github.event.review.user.login }}
          SENDER: ${{ github.event.sender.login }}
        run: |
          request() {
            echo "Requesting review from $2"
            gh api --method POST "repos/$GITHUB_REPOSITORY/pulls/$PR_NUMBER/requested_reviewers" \
              -f "$1[]=$2" --silent || failed=1
          }

          [ "$SENDER" = "$REVIEWER" ] || request reviewers "$REVIEWER"

          teams=$(gh api graphql -f id="$REVIEW_ID" -f query='
            query($id: ID!) {
              node(id: $id) {
                ... on PullRequestReview {
                  onBehalfOf(first: 100) { nodes { slug } }
                  pullRequest {
                    latestOpinionatedReviews(first: 100) {
                      nodes { state author { login } onBehalfOf(first: 100) { nodes { slug } } }
                    }
                  }
                }
              }
            }' --jq '.data.node
              | [.onBehalfOf.nodes[].slug]
                - [.pullRequest.latestOpinionatedReviews.nodes[]
                    | select(.state == "APPROVED" and .author.login != env.REVIEWER)
                    | .onBehalfOf.nodes[].slug]
              | .[]')

          for team in $teams; do
            request team_reviewers "$team"
          done

          exit "${failed:-0}"
```

To re-request only the reviewer, you don't need the app:

```yaml
name: Re-request dismissed reviews

# Like pull_request, this event builds the PR merge ref, so it doesn't fire
# on PRs with merge conflicts.
on:
  pull_request_review:
    types: [dismissed]

permissions:
  pull-requests: write

jobs:
  re-request:
    # Skipped for fork PRs and Dependabot (read-only token on this event) and
    # bot reviewers (can't be requested).
    if: >-
      github.event.pull_request.state == 'open' &&
      github.event.pull_request.head.repo.full_name == github.repository &&
      github.actor != 'dependabot[bot]' &&
      github.event.review.user.type == 'User'
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      # Re-request the reviewer unless they are the sender (they dismissed it,
      # or their push did).
      - name: Re-request review
        env:
          GH_TOKEN: ${{ github.token }}
          PR_NUMBER: ${{ github.event.pull_request.number }}
          REVIEWER: ${{ github.event.review.user.login }}
          SENDER: ${{ github.event.sender.login }}
        run: |
          if [ "$SENDER" != "$REVIEWER" ]; then
            echo "Requesting review from $REVIEWER"
            gh api --method POST "repos/$GITHUB_REPOSITORY/pulls/$PR_NUMBER/requested_reviewers" \
              -f "reviewers[]=$REVIEWER" --silent
          fi
```
