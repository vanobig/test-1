// GetPullRequestCommits gets all commits listed in a pull request
func (service *BitbucketService) GetPullCommits(repo *model.Repo, prModel *model.Pull) ([]*model.PullCommit, error) {
	var commits bitbucket.CommitResponse
	var edges []*model.PullCommit
	var updateKeys []string
	var updateValues []interface{}

	params := map[string]string{
		"username":   repo.Org,
		"repository": repo.Name,
		"id":         strconv.Itoa(*prModel.Number),
	}

	url := BaseBitbucketURL + "/repositories/{username}/{repository}/pullrequests/{id}/commits"
	_, err := service.client.SetPathParams(params).SetResult(&commits).Get(url)
	if err != nil {
		return nil, err
	}

	for _, commit := range commits.Values {
		from := model.NewCommitKey(commit.Hash)
		to := model.NewPullKey(repo.URL, *prModel.Base, *prModel.Head)

		edges = append(edges, model.NewPullCommit(from, to, model.PRDirectionFrom))

		updateKeys = append(updateKeys, from)
		updateValues = append(updateValues, map[string]interface{}{
			"in_pull_request": true,
		})
	}

	if err = service.db.Commit.Update(updateKeys, updateValues); err != nil {
		return nil, err
	}

	return edges, nil
}
