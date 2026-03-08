# Opinions on Build Servers

## Why I prefer to avoid GitHub Actions

GitHub Actions appears to be really popular. GitHub chose the path of "give them what they want" and not "give them what they need." For quick and small projects, a common pain point would be setting up CI and configuring it with a GitHub repo. GitHub managed to swoop in and provide something that was relatively affordable (free in most cases) while being really easy to set up— just a workflow file to your `.github` folder and BAM! you're off to the races.

Was GitHub Actions what we really needed? I would say no. After using it for a few years, I've come across many pitfalls in its design and it has pushed me to lean on other solutions to use as build servers.

Don't get me wrong— I think it is also hard to come by better solutions in the first place. Many of the pitfalls of GitHub Actions are present in the other options including [Travis CI](https://www.travis-ci.com/) and [Circle CI](https://circleci.com/), maybe even [Jekins](https://www.jenkins.io/) depending on how you use them. Albeit, GitHub Actions might stand out due to its "actions marketplace" which other solutions might struggle to offer.

Now that that's said, let's go over the reasons why looks may be deceiving here.

### GitHub Actions turns out to be really insecure

Andrew Nesbitt has done one amongst many great write-ups on this topic. His article [GitHub Actions Has a Package Manager, and It Might Be the Worst](https://nesbitt.io/2025/12/06/github-actions-package-manager) covers the topic in depth. There's not much more to say here. So if you're curious about this, go ahead and read that article.

### GitHub Actions feels clunky

Disclaimer: the clunkiness I'm referring to here is just generally related to how the GitHub UI handles the navigations of status checks for pull requests. This clunkiness could still be present across other third party CI services, as long as GitHub is where you go to check job statuses.

On the surface, the "clunkiness" of GitHub Actions might seem inevitable because the same developer experience is present across other services like GitLab too. It is clunky because, no matter whether you have 1 status check or 10, the flow for navigating from a commit to the logs of a failing job is very mouse-heavy and relies on clicking a bunch of extremely small clickable areas ([Fitt's Law](https://en.wikipedia.org/wiki/Fitts%27s_law)) like clicking on the tiny red "x" icon next to a commit, and/or clicking on a specific status check.

Having logs in the UI is good-enough for most people, but it still makes the developer experience feel second-class compared to working with tools on your machine where you can browse logs with your favorite editor (Vim, VSCode, Terminal, etc.) and make those logs accessible to coding agents like Claude Code.

At the root of all this clunkiness is merely the fact that hooking into the build system lifecycle is not easy at all. If you are someone who wants to set up automated notifications, so that when a build fails, you get a Slack message: good luck. Even if it possible, the complexity involved in setting that up in the first place, and maintaining it, is just too high and it prevents a lot of people from considering to set it up in the first place.

Due to it being really unnatural to "hook" into the build system lifecycle, most developers will just accept the clunky experience of "let it run and come back later." Features like "auto-merge" can kind-of improve the situation here, but it is still far from unlocking the full potential for optimizing developer workflows.

### GitHub Actions feels slow

I'm not an expert here, but based on my experience with GitHub Actions, there are aspects of its design that lead to slower-than-necessary builds. The main factor here is that many GitHub Actions are responsible for installing dependencies and setting up your build server. They are performing actions like fetching installation scripts, running installation scripts, fetching binaries, uncompressing things, changing the system configuration, moving files around, etc.

At the end of the day, if you had done DevOps-related work, you would know that 99% of the work happening in these types of actions is stuff that can be skipped completely if you either A) used a build server with a persistent disk and less isolation with all the requirements pre-installed, or B) set up a hand-crafted docker image which all the requirements pre-installed. The latter, Option B, seems to be the only option when working with GitHub Actions if you ever want to "trim the bloat" on a workflow that has become too slow.

Furthermore, the lag between pushing a commit and being able to start watching the logs is also a problem. Some accept it as an inevitability, but if you know what is it like to do things differently, you can start to see the value in having a tighter developer feedback loop.

# What you've been missing: How a build server could work

Granted, many projects quickly reach a level of scale where their build system is expansive and they need several parallel jobs and they have lots of contributors. So the following approach is far from a one-size-fits-all solution. However, there are also many, many projects that are relatively-small in scope or utilize monorepos with incremental build systems, where having a simplified, consolidated and minimal CI process is in their best interest. That has been the case for most of the projects I have worked on in my career. Especially when you're working on small modules/projects that exist within a bigger ecosystem or a bigger architecture.

Your attention might be spread out across a lot of different things, but at least whenever you find yourself zoomed in on a specific project with its own CI in place, you can have a build server that 1) took very little time to set up, 2) provides an unbelievably fast development feedback loop and 3) makes it trivial to hook into which gives you smooth sailing when shipping fast with a coding agent.

How is this done? Well, if it ain't broke, don't fix it. Meaning, back in the day, we used to just spin up VMs with persistent disks and SSH access, and then we'd just "hack" on them to get them to do what we needed. This was how a lot of professional software was delivered. Not only did SSH into our "boxes" to run builds, but to actual host the production software itself too.

Continuing to stick to this simple "SSH into a box with a persistent disk" approach will likely save you a ton of time compared to GitHub Actions. Need Python? Just "sudo apt install." Need Postgres? "sudo apt install." Something went wrong during a build? Just SSH into the box and start poking around. This saves considerable amounts of time compared to doing a trial-and-error approach by going back and forth with a GitHub Actions workflow. Not to mention, you're not exposing yourself to the same supply chain attacks that come from GitHub Actions package management capabilities.

If you're thinking that setting up a build server from scratch is a lot of work, I'm hoping I can start to change your mind. Here is a all-in-one-script that will set up a pretty-adequate build server with an AWS EC2 t2.micro instance. This requires you to have the "aws" CLI tool installed and configured, but otherwise, something like this should generally be sufficient:

```bash
#!/usr/bin/env bash
set -euo pipefail

KEY_NAME="$(hostname)-key"
SG_NAME="devops-ssh"
SSH_KEY="$HOME/.ssh/id_ed25519"

# 1. Import SSH public key
aws ec2 import-key-pair \
  --key-name "$KEY_NAME" \
  --public-key-material "fileb://${SSH_KEY}.pub"

# 2. Create security group allowing SSH
SG_ID=$(aws ec2 create-security-group \
  --group-name "$SG_NAME" \
  --description "Allow SSH for build server" \
  --query "GroupId" --output text)

aws ec2 authorize-security-group-ingress \
  --group-id "$SG_ID" \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0

# 3. Resolve latest Ubuntu 24.04 AMI
AMI=$(aws ssm get-parameter \
  --name /aws/service/canonical/ubuntu/server/24.04/stable/current/amd64/hvm/ebs-gp3/ami-id \
  --query "Parameter.Value" --output text)

# 4. Launch instance
INSTANCE_ID=$(aws ec2 run-instances \
  --image-id "$AMI" \
  --instance-type t2.micro \
  --key-name "$KEY_NAME" \
  --security-group-ids "$SG_ID" \
  --count 1 \
  --query "Instances[0].InstanceId" --output text)

echo "Waiting for instance $INSTANCE_ID to be running..."
aws ec2 wait instance-running --instance-ids "$INSTANCE_ID"

# 5. Get public IP
PUBLIC_IP=$(aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query "Reservations[0].Instances[0].PublicIpAddress" --output text)

echo "Instance running at $PUBLIC_IP"

# 6. Append SSH config entry
cat >> ~/.ssh/config <<EOF

Host devops
    HostName $PUBLIC_IP
    User ubuntu
    IdentityFile $SSH_KEY
    IdentitiesOnly yes
EOF

# 7. Create bare repo
ssh devops "git init --bare ~/build.git"

# 8. Install pre-receive hook
cat <<'HOOK' | ssh devops "cat > ~/build.git/hooks/pre-receive && chmod +x
~/build.git/hooks/pre-receive"
#!/usr/bin/env bash
set -euo pipefail

while read old_sha new_sha ref; do
    if [ "$new_sha" = "0000000000000000000000000000000000000000" ]; then
        continue
    fi

    echo "--- DevOps: Received push to $ref ($new_sha)"

    unset GIT_QUARANTINE_PATH
    WORK_DIR=$(mktemp -d)
    trap 'rm -rf "$WORK_DIR"' EXIT

    BRANCH="${ref#refs/heads/}"
    echo "--- DevOps: Checking out source for build (branch: $BRANCH, sha: $new_sha)..."
    GIT_WORK_TREE="$WORK_DIR" git checkout -f "$new_sha"

    echo "--- DevOps: Running build..."
    if "$WORK_DIR/build.sh" "$WORK_DIR" "$new_sha" "$ref"; then
        echo "--- DevOps: Build passed. Push accepted."
    else
        echo "--- DevOps: Build FAILED. Push rejected."
        exit 1
    fi

done

exit 0
HOOK

echo "Bare repo created at ~/build.git with pre-receive hook."
```

Now anyone with SSH access can configure their git repo like this:

```bash
git remote add devops devops:build
git push devops main
```

That was less than 200 lines of code. It is not a maintenance nightmare. These are tried-and-tested tools where you're highly unlikely to ever really need to fundamentally change anything going on here. As long as you have password-less sudo access.

If you really want reproducibility, you have a couple of options. One option is to include steps in your build.sh that do things like `if ! command_exists python; then sudo apt install python; fi` so you install stuff that you need and skip the install steps if those things appear to be missing. Another solution is to write a bash script to installing everything in one go. If you want to take this further, you can use an Ansible config. But in general, I promise that working with Ansible configs will be a lot more pleasant of an experience than working with GitHub Actions, due to the easy ability to SSH into your machine and poke around when things aren't working.

## How does this old-school build server compare to GitHub Actions?

I would say, for the projects where it is appropriate (which are many), it improves the developer experience on many levels.

### Speed

You're going to see a much faster build because it becomes much easier to:

1. Prepare dependencies up front; no preparation needed at build-time
2. Aggressive caching (copy entire uncompressed files, symlinking, etc.)
3. Co-located caching (store caches on the same machine, no trips to the network needed)
4. Get a build started as soon as possible
5. Lean on the default cache mechanisms of your build tools (npm, pip, cargo, etc.) that "just work" out of the box

### Security

Not only does it become easier to set up requirements like adding Postgres, Docker, language runtimes, etc., but even the simplest ways to set them up are already safer than relying on off-the-shelf GitHub Actions from untrusted authors. If you're concerned about reproducibility and pinning down specific versions, well it is all up to you to pick a Linux distro with a package manger that has the constraints that you're looking for. In a way, assuming you don't find yourself setting up new build servers from scratch all the time, then you already have a form of version-pinning since you will just install something when setting up the server, and sticking with that specific version indefinitely.

You can also lock down the IP addresses that have access to your build server. This can be a blessing and curse oftentimes. But it should give you more confidence than using GitHub Actions, since it is much harder to spoof an IP than it is to steal an SSH key. There is a lot more to this topic but I'll spare the details for now.

### Developer Experience

My opinion is that the biggest justification for setting up an old-school build server is the developer experience. Running a build is now just using the `git push` command. This allows for developers to naturally hook into the build lifecycle. Developers have the freedom and ability to write scripts that wrap the `git push` in order to run additional logic after it exits, immediately upon exiting. You can use this to send yourself notifications that "just work" with minimal configuration required. You can ask your coding agent "run `git push` please" and now the agent will be able to watch the build and kick into action the moment it finishes.

Another thing worth mentioning here is that the build logs are now in your terminal, it is is super easy to pipe the output to a text file too. So that saves you from all of the steps involved in clicking through the GitHub UI and navigating to the build logs and clicking on the "download" button and then pulling the file out of your downloads folder and so on and so on. I use this all the time in my workflow with NeoVim terminal buffers, by being able to navigate the shell output with Vim commands and then being able to quickly and easily select chunks of text from the build log and send those over to my coding agent without ever leaving the terminal.

When you combine this accessibility, hookability and speed, you start to really see a difference when you're doing things like firefighting and trying to ship bug fixes as fast as possible. You'll see a difference when you're churning on changes to the build system. You'll see a difference when you're making tiny, tiny pull requests and you want to move on quickly to the next task. You'll see a difference when you see a build failure, and you want your coding agent to "just figure it out" while you walk away.

### Maintenance

As counter-intuitive as it may seem, in my experience, the maintenance of a build server has been a lot easier than maintaining a GitHub Actions workflow. Most developers tools are designed to "just work" for normal ubuntu installations and the like, but within GitHub Actions, there are small differences here and there due to containerization or permissions or shell profile files where things don't always "just work" and thus require some customization or searching for an off-the-shelf GitHub Action that has already figured out the necessary customization. All of those situations lead to this trial-and-error effort where you're going back and forth with the build and wafting through the tedious GitHub UI and waiting for builds to finish by just periodically "checking in" instead of getting notified when statuses change.

# Conclusion

I hope this article wasn't too boring. I hope it was interesting and informative. I hope, after reading this, you're more likely to take a chance on setting up a custom build server. If it is truly _really fast_ to set up, then what do you have to lose? If you can get something working in 5-10 minutes, then worst case scenario, at some point later on, you'll discover that you've wasted 5-10 minutes and then you'll proceed to go and set up your first GitHub Actions workflow file.

Try it out! Chances are that the hardest part is to explain to your team why you're doing this, and getting them to set up SSH access and remember to do `git push devops ...` or something like that.
