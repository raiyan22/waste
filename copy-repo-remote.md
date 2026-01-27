to clone a remote repo to another remote repo,  create new repo ion github and then:

### Navigate to script location
`cd ~/Documents`

#### Run script
`./copy-repo.sh <source> <dest>`

#### Answer prompts: 
`yes → yes → PUSH → yes `

note: we might need to use `chmod +x` 

keeping the `copy-repo.sh` in Documents and it contains: 

```
#!/bin/bash

# ============================================
# Safe Repo Copy Script
# Usage: ./copy-repo.sh <source-repo> <dest-repo>
# ============================================

# Check if arguments provided
if [ $# -ne 2 ]; then
    echo "Usage: $0 <source-repo-url> <dest-repo-url>"
    echo ""
    echo "Example:"
    echo "  $0 https://github.com/rai-wtag/wnlpp-copy.git https://github.com/rai-wtag/revov-poc02.git"
    exit 1
fi

# Configuration from arguments
SOURCE_REPO="$1"
DEST_REPO="$2"
TEMP_DIR="$HOME/Documents/temp-repo-copy"

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

echo "============================================"
echo "        SAFE REPO COPY SCRIPT"
echo "============================================"
echo ""
echo -e "Source:      ${YELLOW}$SOURCE_REPO${NC}"
echo -e "Destination: ${GREEN}$DEST_REPO${NC}"
echo ""

# Confirmation 1
read -p "Is this correct? (yes/no): " confirm1
if [ "$confirm1" != "yes" ]; then
    echo -e "${RED}Aborted.${NC}"
    exit 1
fi

# Create temp directory
echo ""
echo "Creating temp directory: $TEMP_DIR"
mkdir -p "$TEMP_DIR"
cd "$TEMP_DIR" || exit 1

# Remove existing clone if present
if [ -d "repo-clone" ]; then
    echo "Removing existing clone..."
    rm -rf repo-clone
fi

# Clone source repo
echo ""
echo "Cloning source repo..."
git clone "$SOURCE_REPO" repo-clone

if [ $? -ne 0 ]; then
    echo -e "${RED}Clone failed!${NC}"
    exit 1
fi

cd repo-clone || exit 1

# Show contents
echo ""
echo "============================================"
echo "Cloned contents:"
echo "============================================"
ls -la
echo ""

# Safety check - show current remote
echo "============================================"
echo "Current remote (BEFORE change):"
echo "============================================"
git remote -v
echo ""

# Confirmation 2
read -p "Proceed to change remote to destination? (yes/no): " confirm2
if [ "$confirm2" != "yes" ]; then
    echo -e "${RED}Aborted.${NC}"
    exit 1
fi

# Remove origin and add new one
git remote remove origin
git remote add origin "$DEST_REPO"

# Safety check - show new remote
echo ""
echo "============================================"
echo -e "${GREEN}NEW remote (AFTER change):${NC}"
echo "============================================"
git remote -v
echo ""

# Final confirmation before push
echo -e "${YELLOW}⚠️  FINAL CHECK BEFORE PUSH ⚠️${NC}"
echo ""
echo -e "You are about to push to: ${GREEN}$DEST_REPO${NC}"
echo -e "Source repo ${YELLOW}$SOURCE_REPO${NC} will NOT be modified."
echo ""
read -p "Type 'PUSH' to confirm: " confirm3
if [ "$confirm3" != "PUSH" ]; then
    echo -e "${RED}Aborted. Nothing was pushed.${NC}"
    exit 1
fi

# Push all branches and tags
echo ""
echo "Pushing all branches..."
git push -u origin --all

echo ""
echo "Pushing all tags..."
git push origin --tags

# Success
echo ""
echo "============================================"
echo -e "${GREEN}✅ SUCCESS!${NC}"
echo "============================================"
echo -e "Contents copied to: ${GREEN}$DEST_REPO${NC}"
echo -e "Source unchanged:   ${YELLOW}$SOURCE_REPO${NC}"
echo ""
echo "Temp files location: $TEMP_DIR/repo-clone"
read -p "Delete temp files? (yes/no): " cleanup
if [ "$cleanup" == "yes" ]; then
    cd "$HOME"
    rm -rf "$TEMP_DIR"
    echo "Cleaned up!"
fi

echo ""
echo "Done! 🎉"

```
