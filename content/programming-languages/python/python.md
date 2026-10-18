# Python

## References

* [Python 2.7: Unicode HOWTO](https://docs.python.org/2.7/howto/unicode.html)

## Usage

### List: One line expression

```python
my_list = [1, 2, 3]
print([ n for n in my_list if n == 1 ][0]) # 1
```

### Print in one line dynamically

- Print dots in one line (for example, in "for" iterations)

```python
for item in items:
    print('.', sep='', end='', flush=True)
```

### Save JSON to file

- Save JSON to file

```python
with open('file_name.json', 'w+', encoding='utf-8') as outfile:
    outfile.write(json.dumps(json_data, indent=4, ensure_ascii=False))
```

### Create requirements.txt

- Install pip and use freeze command

```bash
pip freeze > requirements.txt
```

### Create requirements.txt from directory

- Install pipreqs

```bash
pip install pipreqs
```

- Create requirements.txt from directory

```bash
pipreqs /home/project/location
```
