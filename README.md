# pytest-cover

[![made-with-python](https://img.shields.io/badge/Made%20with-Python-1f425f.svg)](https://www.python.org)

The repository [pytester-cov](https://github.com/alexanderdamiani/pytester-cov) was taken as a basis.

## Python Packages Used

- [`pytest`](https://pypi.org/project/pytest/)
- [`pytest-cov`](https://pypi.org/project/pytest-cov/)

## Required Inputs

- `python-version`
  - Version of python interpreter you use in your project

## Optional Inputs


- `package-manager`
  - Python package manager you use in your project
  - `pip` by default 

- `pytest-root-dir`
  - root directory to recursively search for .py files
  - by default `pytest --cov` does not run recursively, but will here
- `pytest-tests-dir`
  - directory with pytest tests
  - if left empty will identify test(s) dir by default
- `requirements-file`
  - requirements filepath for project
  - if left empty will default to `requirements.txt`
  - necessary if you use `pip` python package manager
- `cov-omit-list`
  - list of directories and/or files to ignore
- `cov-threshold-single`
  - fail if any single file coverage is less than threshold
- `cov-threshold-total`
  - fail if the total coverage is less than threshold

## Outputs

- `output-table`
  - str
  - `pytest --cov` markdown output table
- `cov-threshold-single-fail`
  - boolean
  - `false` if any single file coverage less than `cov-threshold-single`, else `true`
- `cov-threshold-total-fail`
  - boolean
  - `false` if total coverage less than `cov-threshold-total`, else `true`

## Template workflow file

```yaml
name: pytester-cov workflow

on: [pull_request]

jobs:
  tests:

    runs-on: ubuntu-latest
    env:
      PYTHON_VERSION: 3.9.6
      PACKAGE_MANAGER: poetry
      COVERAGE_SINGLE: 60
      COVERAGE_TOTAL: 60
      PULL_NUMBER: ${{ github.event.pull_request.number }}
      COMMIT_URL: "https://github.com/${{ github.repository }}/pull/${{ github.event.pull_request.number }}/commits/${{ github.event.after }}"
    steps:
    - name: pytester-cov
      id: pytester-cov
      uses: dima-engineer/pytest-cover@main
      with:
        python-version: ${{ env.PYTHON_VERSION }}
        package-manager: ${{ env.PACKAGE_MANAGER }}
        pytest-root-dir: '.'
        cov-omit-list: 'test/*, temp/main3.py, temp/main4.py'
        cov-threshold-single: ${{ env.COVERAGE_SINGLE }}
        cov-threshold-total: ${{ env.COVERAGE_TOTAL }}
    
    - name: Find Comment
      uses: peter-evans/find-comment@v1
      id: fc
      with:
        issue-number: ${{env.PULL_NUMBER}}
        comment-author: 'github-actions[bot]'
        direction: last

    - name: Add SHORT_SHA env property with commit short sha
      run: echo "SHORT_SHA=`echo ${{github.event.after}} | cut -c1-7`" >> $GITHUB_ENV
    
    - name: Create comment with pytest coverage table
      uses: peter-evans/create-or-update-comment@v1
      with:
        issue-number: ${{env.PULL_NUMBER}}
        comment-id: ${{ steps.fc.outputs.comment-id }}
        body: |
          ## Tests coverage table for [${{ env.SHORT_SHA }}](${{ env.COMMIT_URL }}) commit.
          ${{ steps.pytester-cov.outputs.output-table }}
        edit-mode: replace
    
    - name: Coverage single fail - new issue
      if: ${{ steps.pytester-cov.outputs.cov-threshold-single-fail == 'true' }}
      uses: nashmaniac/create-issue-action@v1.1
      with:
        title: Pytest coverage single falls below minimum ${{ env.COVERAGE_SINGLE }}
        token: ${{secrets.GITHUB_TOKEN}}
        assignees: ${{github.actor}}
        labels: workflow-failed
        body: ${{ steps.pytester-cov.outputs.output-table }}

    - name: Coverage single fail - exit
      if: ${{ steps.pytester-cov.outputs.cov-threshold-single-fail == 'true' }}
      run: |
        echo "cov single fail ${{ steps.pytester-cov.outputs.cov-threshold-single-fail }}"
        exit 1

    - name: Coverage total fail - new issue
      if: ${{ steps.pytester-cov.outputs.cov-threshold-total-fail == 'true' }}
      uses: nashmaniac/create-issue-action@v1.1
      with:
        title: Pytest coverage total falls below minimum ${{ env.COVERAGE_TOTAL }}
        token: ${{secrets.GITHUB_TOKEN}}
        assignees: ${{github.actor}}
        labels: workflow-failed
        body: ${{ steps.pytester-cov.outputs.output-table }}

    - name: Coverage total fail - exit
      if: ${{ steps.pytester-cov.outputs.cov-threshold-total-fail == 'true' }}
      run: |
        echo "cov single fail ${{ steps.pytester-cov.outputs.cov-threshold-total-fail }}"
        exit 1

    - name: Commit pytest coverage table
      uses: peter-evans/commit-comment@v1
      with:
        body: ${{ steps.pytester-cov.outputs.output-table }}
```
