# Objectives
In this lab, you will learn how to perform the following tasks: 
    Create a new repository
    Add a Google Source Repository as a remote
    Push to the Cloud Source Repository

# Task 1. Create a new repository

1. Start a new session in Cloud Shell and run the following command to create a new Cloud Source Repository named REPO_DEMO:
    gcloud source repos create REPO_DEMO

# Task 2. Clone the new repository into your Cloud Shell session

1. Clone the contents of your new Cloud Source Repository to a local repo in your Cloud Shell session:
    gcloud source repos clone REPO_DEMO

    The gcloud source repos clone command adds Cloud Source Repositories as a remote named origin and clones it into a local Git repository

# Task 3. Push to the Cloud Source Repository

1. Go into the local repository you created:
    cd REPO_DEMO

2. Run the following command to create a file myfile.txt in your local repository:
    echo 'Hello World!' > myfile.txt

3. Commit the file using the following Git commands:
    git config --global user.email "bishaldhimal321@gmail.com"
    git config --global user.name "Bishal"
    git add myfile.txt
    git commit -m "First file using Cloud Source Repositories" myfile.txt

4.  Once you've committed code to the local repository, add its contents to Cloud Source Repositories using the git push command:
    git push origin master

5. Git pushes the sample application files from the master branch to the origin remote:


# Task 4. Browse files in the Google Cloud Source Repository

1. Run the command to list the Repositories:
    gcloud source repos list

# Task 5. View a file in the Google Cloud repository
1. In the Console go to Navigation menu > Source Repositories.



# history
1  gcloud auth list
2  gcloud config list project
3  gcloud config set project qwiklabs-gcp-01-e224a8101ba7
4  gcloud source repos create REPO_DEMO
5  gcloud source repos clone REPO_DEMO
6  ls
7  cd REPO_DEMO
8  echo 'Hello World!' > myfile.txt
9  git config --global user.email "bishaldhimal321@gmail.com"
10  git config --global user.name "Bishal"
11  git add myfile.txt
12  git commit -m "First file using Cloud Source Repositories" myfile.txt
13  git push origin master
14  gcloud source repos list
15  history