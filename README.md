cat >> ~/.bashrc << 'EOF'

sync-push() {
  cd ~/nixos-config
  git add -A
  git commit -m "${1:-sync $(date +%Y-%m-%d_%H:%M)}"
  git push
}

sync-pull() {
  cd ~/nixos-config
  git pull
  nixos-rebuild switch --flake .#laptop
}
EOF
source ~/.bashrc   
