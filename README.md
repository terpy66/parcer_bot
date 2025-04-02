import csv

import pandas as pd
import requests
from bs4 import BeautifulSoup
from fake_useragent import UserAgent
from tabulate import tabulate
import re
from urllib3.filepost import writer


URL = "https://www.work.ua/jobs-remote-it/"
UA = UserAgent()


def get_html(url):
    headers = {"User-Agent": UA.random}
    response = requests.get(url, headers=headers)

    if response.status_code == 200:
        return response.text
    return None


def get_total_pages(html):
    soup = BeautifulSoup(html, "html.parser")
    pagination = soup.find("ul", class_="pagination")

    if pagination:
        pages = pagination.find_all("a")
        if len(pages) > 1:
            last_page = pages[-2].get_text(strip=True)
            return int(last_page)

    return 1


def parse_jobs(html):
    soup = BeautifulSoup(html, "html.parser")
    jobs = set()  # Используем set, чтобы избежать дубликатов
    blocks = soup.find_all("div", tabindex="0")

    for block in blocks:
        name = block.find("a", tabindex="-1")
        company = block.find("div", class_="mt-xs")
        desc = block.find("p", class_="ellipsis ellipsis-line ellipsis-line-3 text-default-7 mb-0")
        salary = block.find("span", class_="strong-600")

        name_text = name.get_text(strip=True) if name else "Не вказано"
        company_text = company.get_text(strip=True) if company else "Не вказано"
        desc_text = desc.get_text(strip=True) if desc else "Не вказано"
        salary_text = salary.get_text(strip=True) if salary else None

        company_text = re.sub(r"\s*\(?Дистанційно\)?", " ", company_text)
        company_text = re.sub(r"\s*\(?Вся Україна\)?", " ", company_text)


        job_tuple = (name_text, company_text, salary_text, desc_text)  

        jobs.add(job_tuple) 

    return list(jobs) e


def save_to_csv(jobs):
    df = pd.DataFrame(jobs, columns=["Название","Компания","Зарплата","Описание"])
    df.to_csv("data.csv", index=False, encoding="utf-8-sig")
def print_table(jobs):
    table = tabulate(jobs,  headers=["Название","Компания","Зарплата","Описание"], tablefmt="grid")
    print(table)


def main():
    html = get_html(URL)
    if not html:
        print("Помилка отримання HTML-коду сторінки")
        return

    total_pages = get_total_pages(html)


    all_jobs = set()  

    for page in range(1, total_pages + 1):
        page_url = f"{URL}?page={page}"
        html = get_html(page_url)

        if html:
            jobs = parse_jobs(html)
            all_jobs.update(jobs)  

    all_jobs_list = list(all_jobs) 
    save_to_csv(all_jobs_list)
    print_table(all_jobs_list)
    print("all good")




if __name__ == "__main__":
    main()
