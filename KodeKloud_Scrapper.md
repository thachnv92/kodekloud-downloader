# Disclaimer
Please use this only to keep your local copy. Please don't distribute the courses illegally. You will be responsible for the legal consequences. 

---

# Steps to follow
- Sign in to kodekloud.com
- Right Click -> Inspect -> Go to Network Tab -> Reload -> Select the first resource and copy the cookie:
- ![](https://i.imgur.com/MwkR9u6.png)
- Install Python and then some depencencies (use virtual environment if possible):
`pip install requests yt-dlp beautifulsoup4 markdownify`
- Use this code to download all video lecture for all courses (make sure to replace cookie and download path):

<details>
<summary>python kodekloud_scrap.py</summary>
  
```python
import string
import subprocess
from dataclasses import dataclass
from pathlib import Path
from shlex import split
from typing import List

import markdownify
import requests
import yt_dlp
from bs4 import BeautifulSoup

all_course_link = [
    "https://kodekloud.com/courses/docker-for-the-absolute-beginner/",
    "https://kodekloud.com/courses/red-hat-certified-system-administratorrhcsa-2/",
    "https://kodekloud.com/courses/kustomize/",
    "https://kodekloud.com/courses/devsecops/",
    "https://kodekloud.com/courses/az-104-microsoft-azure-administrator/",
    "https://kodekloud.com/courses/devops-interview-prep-course/",
    "https://kodekloud.com/courses/terraform-challenges/",
    "https://kodekloud.com/courses/hashicorp-certified-consul-associate-certification/",
    "https://kodekloud.com/courses/hashicorp-certified-vault-associate-certification/",
    "https://kodekloud.com/courses/kubernetes-challenges/",
    "https://kodekloud.com/courses/linux-challenges/",
    "https://kodekloud.com/courses/hashicorp-certified-vault-operations-professional-2022/",
    "https://kodekloud.com/courses/cks-challenges/",
    "https://kodekloud.com/courses/linux-foundation-certified-system-administrator-lfcs/",
    "https://kodekloud.com/courses/jenkins/",
    "https://kodekloud.com/courses/golang/",
    "https://kodekloud.com/courses/hashicorp-certified-terraform-associate/",
    "https://kodekloud.com/courses/helm-for-beginners/",
    "https://kodekloud.com/courses/certified-associate-in-python-programming/",
    "https://kodekloud.com/courses/istio-service-mesh/",
    "https://kodekloud.com/courses/jinja2-templating/",
    "https://kodekloud.com/courses/certified-kubernetes-security-specialist-cks/",
    "https://kodekloud.com/courses/python-entry-level-programmer-certification/",
    "https://kodekloud.com/courses/json-path-quiz/",
    "https://kodekloud.com/courses/docker-certified-associate-exam-course/",
    "https://kodekloud.com/courses/terraform-for-beginners/",
    "https://kodekloud.com/courses/certified-kubernetes-administrator-cka/",
    "https://kodekloud.com/courses/certified-kubernetes-application-developer-ckad/",
    "https://kodekloud.com/courses/kubernetes-for-the-absolute-beginners-hands-on/",
    "https://kodekloud.com/courses/chef-for-the-absolute-beginners/",
    "https://kodekloud.com/courses/openshift-for-the-absolute-beginners/",
    "https://kodekloud.com/courses/ansible-certification-preparation-course/",
    "https://kodekloud.com/courses/ansible-for-the-absolute-beginners-couse/",
    "https://kodekloud.com/courses/docker-swarm-services-stacks-hands-on/",
    "https://kodekloud.com/courses/devops-pre-requisite-course/",
    "https://kodekloud.com/courses/the-linux-basics-course/",
    "https://kodekloud.com/courses/shell-scripts-for-beginners/",
    "https://kodekloud.com/courses/puppet-for-the-absolute-beginners-course/",
    "https://kodekloud.com/courses/git-for-beginners/",
    "https://kodekloud.com/courses/aws-cloud-for-beginners/",
    "https://kodekloud.com/courses/lens-introduction/",
    "https://kodekloud.com/courses/linode-kubernetes-engine/",
]


@dataclass
class Lesson:
    name: str
    url: str
    is_video: bool


@dataclass
class Topic:
    name: str
    lessons: List[Lesson]

    @classmethod
    def make(cls, topic):
        title = topic.find("div", class_="ld-item-title")
        name = title.span.text.strip()

        lessons_urls = topic.find_all("a", class_="ld-topic-row")
        lessons_names = topic.find_all("span", class_="ld-topic-title")

        lessons = []

        for lesson_url, lesson_name in zip(lessons_urls, lessons_names):
            is_video = "video_topic_enabled" in lesson_name["class"]
            lessons.append(
                Lesson(
                    name=lesson_name.text.strip(),
                    url=lesson_url["href"],
                    is_video=is_video,
                )
            )
        return Topic(name=name, lessons=lessons)


# def download_video(url, output_path):
#     yt_dlp.utils.std_headers[
#         "Cookie"
#     ] = """"""
#     ydl_opts = {
#         "concurrent_fragment_downloads": 15,
#         "outtmpl": f"{output_path}.%(ext)s",
#     }
#     with yt_dlp.YoutubeDL(ydl_opts) as ydl:
#         ydl.download([url])

COOKIE = """<REPLACE_WITH_YOUR_COOKIE>"""


def download_video(url: str, output_path: Path) -> int:
    command = f"yt-dlp -f bestvideo+bestaudio -N 15 --add-header 'Cookie:{COOKIE}' -o '{output_path}' {url}"
    process = subprocess.Popen(split(command))
    return process.wait()


def normalize_name(name: str) -> str:
    return name.translate(str.maketrans("", "", string.punctuation))


def is_normal_content(content: str) -> bool:
    is_lab = content.find("div", class_="start-lab-button")
    is_feedback = content.find_all("iframe")
    return not (is_lab or is_feedback)


def download_all_pdf(content, download_path: Path) -> None:
    for link in content.find_all("a"):
        href = link.get("href")
        if href.endswith("pdf"):
            file_name = download_path / Path(href).name
            print(f"Downloading {file_name}...")
            response = requests.get(href, headers={"Cookie": COOKIE})
            file_name.write_bytes(response.content)


for course_link in all_course_link:
    page = requests.get(course_link)
    soup = BeautifulSoup(page.content, "html.parser")

    topics = soup.find_all("div", class_="ld-item-list-item")
    course_title = soup.find("h1", class_="entry-title").text.strip()
    items = []
    for topic in topics:
        items.append(Topic.make(topic))
    for i, item in enumerate(items, start=1):
        for j, lesson in enumerate(item.lessons, start=1):
            file_path = Path(
                Path.home()
                / "Downloads"
                / "KodeKloud"
                / normalize_name(course_title)
                / f"{i} - {normalize_name(item.name)}"
                / f"{j} - {normalize_name(lesson.name)}"
            )
            if lesson.is_video:
                print(f"Writing video file... {file_path}...")
                file_path.parent.mkdir(parents=True, exist_ok=True)
                download_video(lesson.url, file_path.with_suffix(".mp4"))
            else:
                page = requests.get(lesson.url, headers={"Cookie": COOKIE})
                soup = BeautifulSoup(page.content, "html.parser")
                content = soup.find("div", class_="learndash_content_wrap")
                if is_normal_content(content):
                    print(f"Writing resource file... {file_path}...")
                    file_path.parent.mkdir(parents=True, exist_ok=True)
                    file_path.with_suffix(".md").write_text(
                        markdownify.markdownify(content.prettify()), encoding="utf-8"
                    )
                    download_all_pdf(content, file_path.parent)
```
</details>

**NOTE:**
For some courses we need to enroll first before downloading. Example:
![image](https://user-images.githubusercontent.com/17507690/192079208-099388cd-6a03-4573-b71a-96cdd8b4fff6.png)

If we don't enroll then it download same introduction video for all the lessons.

---

# Course Decks

[Gist](https://gist.github.com/debakarr/533e4140314bc252f04b04afc02dd20c)
