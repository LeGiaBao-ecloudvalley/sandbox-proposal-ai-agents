### Run on CMD
```sh 
> python3 -m venv <your-virtual-env-name>
```

Example:
```sh 
> python3 -m venv myenv
```

### Activate the virtual env (Windows)
```sh 
> .\<your-virtual-env-name>\Scripts\activate
```

### Add the requirements.txt to your repo
`requirements.txt`
```txt
streamlit>=1.28.0
mcp>=0.9.0
boto3>=1.34.0
langchain-aws>=0.1.0
langchain-core>=0.1.0
```

### Install dependencies
```sh 
> python -m pip install -r requirements.txt
```