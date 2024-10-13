# get values.yaml
helm show values gitlab/gitlab-runner --version 0.69.0 > value_gitlab-runner.yaml
helm show values gitlab/gitlab-runner > values_gitlab-runner.yaml

# pull whole chart
# helm pull repo/chartname [flags]
# --untar: untar the chart after downloading it
helm pull gitlab/gitlab-runner --untar

helm install --namespace gitlab --create-namespace gitlab-runner gitlab/gitlab-runner  -f values_gitlab-runner.yaml


## Reference
* [](https://www.alibabacloud.com/help/tc/ack/use-gitlab-ci-to-run-a-gitlab-runner-and-run-a-pipeline-on-kubernetes#section-nmt-nsp-qgb)