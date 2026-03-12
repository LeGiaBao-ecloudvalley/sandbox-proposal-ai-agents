## Requirements

#### Installation:

##### chrome-devtools-mcp
```
# Install the chrome-devtools-mcp server
npm install -g chrome-devtools-mcp
```

The chrome-devtools-mcp package connects to Chrome via the Chrome DevTools Protocol (CDP) and requires Chrome to be running with remote debugging enabled.

```
# Windows, run in CMD:
> "C:\Program Files\Google\Chrome\Application\chrome.exe" --remote-debugging-port=9222 --user-data-dir="C:\chrome-debug-profile"
```


##### MCP client dependencies
`requirements.txt`
```txt
streamlit>=1.28.0
mcp>=0.9.0
boto3>=1.34.0
langchain-aws>=0.1.0
langchain-core>=0.1.0
```
> pip install -r requirements.txt


##### MCP client code
```python
# client.py
import streamlit as st
import asyncio
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client


from langchain_core.messages import HumanMessage
import json
import os
import boto3
from botocore.config import Config
from langchain_aws import ChatBedrockConverse

region = os.environ.get('AWS_REGION', 'us-west-2') or None
helicone_bedrock_endpoint = f"https://bedrock.helicone.ai/v1/{region}"

bedrock_config = Config(
    retries={
        'max_attempts': 5,
        'mode': 'adaptive'
    },
    read_timeout=3600,
)

bedrock_runtime_client = boto3.client(
    service_name="bedrock-runtime",
    region_name=region,
    endpoint_url=helicone_bedrock_endpoint,
    config=bedrock_config
)

client = ChatBedrockConverse(
    bedrock_client=bedrock_runtime_client,
    provider='anthropic',
    model_id='anthropic.claude-3-5-sonnet-20241022-v2:0',
    region_name=region,
    temperature=0,
    max_tokens=4096,
    config=bedrock_config
)

st.set_page_config(page_title="Chrome DevTools MCP Client", page_icon="🌐")
st.title("🌐 Chrome DevTools MCP Client")

# Initialize session state
if 'mcp_session' not in st.session_state:
    st.session_state.mcp_session = None
if 'tools' not in st.session_state:
    st.session_state.tools = []
if 'connected' not in st.session_state:
    st.session_state.connected = False

async def connect_to_chrome_mcp(chrome_port=9222):
    """Connect to chrome-devtools-mcp server"""
    server_params = StdioServerParameters(
        command="npx",
        args=[
            "chrome-devtools-mcp",
            # The server will connect to Chrome at localhost:9222 by default
        ],
        env=None
    )
    
    try:
        # Note: Using context manager for the connection
        read, write = await stdio_client(server_params).__aenter__()
        session = await ClientSession(read, write).__aenter__()
        
        # Initialize the connection
        await session.initialize()
        
        # List available tools
        tools_response = await session.list_tools()
        
        return session, tools_response.tools
    except Exception as e:
        st.error(f"Failed to connect: {e}")
        return None, []

async def call_tool(session, tool_name, arguments):
    """Call an MCP tool"""
    try:
        result = await session.call_tool(tool_name, arguments=arguments)
        return result
    except Exception as e:
        return {"error": str(e)}

# Sidebar configuration
with st.sidebar:
    st.header("⚙️ Configuration")
    
    chrome_port = st.number_input(
        "Chrome Debug Port",
        value=9222,
        help="Chrome must be running with --remote-debugging-port"
    )
    
    if st.button("🔌 Connect to Chrome MCP", type="primary"):
        with st.spinner("Connecting to chrome-devtools-mcp..."):
            try:
                session, tools = asyncio.run(connect_to_chrome_mcp(chrome_port))
                if session:
                    st.session_state.mcp_session = session
                    st.session_state.tools = tools
                    st.session_state.connected = True
                    st.success(f"✅ Connected! Found {len(tools)} tools")
                    st.rerun()
            except Exception as e:
                st.error(f"❌ Connection failed: {e}")
    
    if st.session_state.connected:
        st.success("🟢 Connected")
        
        if st.button("🔌 Disconnect"):
            st.session_state.connected = False
            st.session_state.mcp_session = None
            st.session_state.tools = []
            st.rerun()
    else:
        st.warning("🔴 Not connected")
    
    # Show available tools
    if st.session_state.tools:
        st.divider()
        st.subheader("🛠️ Available Tools")
        for tool in st.session_state.tools:
            with st.expander(f"📌 {tool.name}"):
                st.write(tool.description)
                if hasattr(tool, 'inputSchema'):
                    st.json(tool.inputSchema)

# Main content area
if not st.session_state.connected:
    st.info("👈 Please connect to Chrome MCP using the sidebar")
    
    st.markdown("""
    ### 🚀 Getting Started
    
    1. **Start Chrome with debugging:**
       ```bash
       chrome.exe --remote-debugging-port=9222 --user-data-dir="C:\\chrome-debug-profile"
       ```
    
    2. **Install chrome-devtools-mcp:**
       ```bash
       npm install -g chrome-devtools-mcp
       ```
    
    3. **Click 'Connect to Chrome MCP'** in the sidebar
    
    ### 📚 Common Tools
    - Navigate to URLs
    - Take screenshots
    - Execute JavaScript
    - Click elements
    - Fill forms
    - Get page content
    """)
else:
    # Create tabs for different functionalities
    tab1, tab2, tab3, tab4 = st.tabs(["🌐 Browser Control", "📸 Screenshots", "🔧 Direct Tool Call", "🤖 AI Assistant"])
    
    with tab1:
        st.subheader("Browser Navigation & Control")
        
        col1, col2 = st.columns([3, 1])
        with col1:
            url = st.text_input("URL to navigate", value="https://www.example.com")
        with col2:
            if st.button("🔗 Navigate", type="primary"):
                with st.spinner(f"Navigating to {url}..."):
                    result = asyncio.run(
                        call_tool(st.session_state.mcp_session, "navigate", {"url": url})
                    )
                    st.json(result)
        
        col1, col2, col3 = st.columns(3)
        with col1:
            if st.button("⬅️ Back"):
                result = asyncio.run(
                    call_tool(st.session_state.mcp_session, "goBack", {})
                )
                st.json(result)
        with col2:
            if st.button("➡️ Forward"):
                result = asyncio.run(
                    call_tool(st.session_state.mcp_session, "goForward", {})
                )
                st.json(result)
        with col3:
            if st.button("🔄 Reload"):
                result = asyncio.run(
                    call_tool(st.session_state.mcp_session, "reload", {})
                )
                st.json(result)
    
    with tab2:
        st.subheader("Take Screenshot")
        
        if st.button("📸 Capture Screenshot"):
            with st.spinner("Taking screenshot..."):
                result = asyncio.run(
                    call_tool(st.session_state.mcp_session, "screenshot", {})
                )
                if result and not result.get("error"):
                    st.success("Screenshot taken!")
                    st.json(result)
                else:
                    st.error(result)
    
    with tab3:
        st.subheader("Direct Tool Call")
        
        # Tool selector
        tool_names = [tool.name for tool in st.session_state.tools]
        selected_tool = st.selectbox("Select Tool", tool_names)
        
        # Show tool details
        tool_info = next((t for t in st.session_state.tools if t.name == selected_tool), None)
        if tool_info:
            st.write(f"**Description:** {tool_info.description}")
            
            # Arguments input
            st.write("**Arguments (JSON):**")
            arguments_json = st.text_area(
                "Tool Arguments",
                value="{}",
                height=150,
                help="Enter arguments as JSON"
            )
            
            if st.button("🚀 Execute Tool"):
                try:
                    arguments = json.loads(arguments_json)
                    with st.spinner(f"Calling {selected_tool}..."):
                        result = asyncio.run(
                            call_tool(st.session_state.mcp_session, selected_tool, arguments)
                        )
                        st.success("Tool executed!")
                        st.json(result)
                except json.JSONDecodeError:
                    st.error("Invalid JSON in arguments")
                except Exception as e:
                    st.error(f"Error: {e}")

        with tab4:
            st.subheader("AI-Powered Browser Control")
            
            user_prompt = st.chat_input("Tell the AI what to do with the browser...")
            
            if user_prompt:
                # Convert MCP tools to Claude format
                claude_tools = []
                for tool in st.session_state.tools:
                    claude_tools.append({
                        "name": tool.name,
                        "description": tool.description,
                        "input_schema": tool.inputSchema
                    })
                
                # Call Claude with tools
                response = client.invoke(
                    [HumanMessage(content=user_prompt)],
                    tools=claude_tools
                )
                
                # Handle tool calls
                if response.stop_reason == "tool_use":
                    for block in response.content:
                        if block.type == "tool_use":
                            # AI decided to call a tool
                            result = asyncio.run(
                                call_tool(
                                    st.session_state.mcp_session,
                                    block.name,
                                    block.input
                                )
                            )
                            st.write(f"🤖 Called: {block.name}")
                            st.json(result)
```

### Run streamlit MCP client
> streamlit run client.py