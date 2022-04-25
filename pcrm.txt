pragma solidity ^0.5.0;

interface IERC20 {
    function totalSupply() external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
    function transfer(address recipient, uint256 amount) external returns (bool);
    function allowance(address owner, address spender) external view returns (uint256);
    function approve(address spender, uint256 amount) external returns (bool);
    function transferFrom(address sender, address recipient, uint256 amount) external returns (bool);
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
}
library SafeMath {

    function add(uint256 a, uint256 b) internal pure returns (uint256) {
        uint256 c = a + b;
        require(c >= a, "SafeMath: addition overflow");
        return c;
    }

    function sub(uint256 a, uint256 b) internal pure returns (uint256) {
        require(b <= a, "SafeMath: subtraction overflow");
        uint256 c = a - b;
        return c;
    }

    function mul(uint256 a, uint256 b) internal pure returns (uint256) {
        if (a == 0) {
            return 0;
        }
        uint256 c = a * b;
        require(c / a == b, "SafeMath: multiplication overflow");
        return c;
    }

    function div(uint256 a, uint256 b) internal pure returns (uint256) {
        require(b > 0, "SafeMath: division by zero");
        uint256 c = a / b;
        return c;
    }

    function mod(uint256 a, uint256 b) internal pure returns (uint256) {
        require(b != 0, "SafeMath: modulo by zero");
        return a % b;
    }
}

contract PCRMKIP7 is IERC20 {
    using SafeMath for uint256;
    address _owner;
    mapping (address => uint256) private _balances;
    mapping (address => mapping (address => uint256)) private _allowances;

    struct lock_box{
        uint256 value_;
        uint256 releaseTime;
    }
    
    mapping(address => bool) private _frozen;
    mapping(address => lock_box[]) internal lockbox_arr;
    mapping(address => uint256) internal totalLock;

    string private _name;
    string private _symbol;
    uint8 private _decimals;
    constructor (string memory name, string memory symbol, uint8 decimals) public {
        _name = name;
        _symbol = symbol;
        _decimals = decimals;
        _mint(msg.sender, 3500000000 * 10 ** uint256(decimals)); // 주의!
        _owner = msg.sender;
    }

    modifier onlyOwner {
      require(msg.sender == _owner);
      _;
    }

    modifier chkFrozen(address target) {
        require(!_frozen[target], "Frozen: account is frozen");
        _;
    }
    modifier chkLocked(address from, uint256 amount) {
        require(_balances[from] > totalLock[from].add(amount), "Lock: There is not enough unlocked quantity");
        _;
    }

    event Freeze(address indexed target);
    event Unfreeze(address indexed target);
    event Lock(address indexed account, uint256 value_, uint256 releaseTime);
    event Unlock(address indexed account, uint256 value_);

    function name() public view returns (string memory) {
        return _name;
    }

    function symbol() public view returns (string memory) {
        return _symbol;
    }

    function decimals() public view returns (uint8) {
        return _decimals;
    }

    uint256 private _totalSupply;

    function totalSupply() public view returns (uint256) {
        return _totalSupply;
    }

    function balanceOf(address account) public view returns (uint256) {
        return _balances[account];
    }

    function transfer(address recipient, uint256 amount) public chkFrozen(msg.sender) chkLocked(msg.sender, amount) returns (bool) {
        _transfer(msg.sender, recipient, amount);
        return true;
    }

    function allowance(address owner, address spender) public view returns (uint256) {
        return _allowances[owner][spender];
    }

    function approve(address spender, uint256 value) public returns (bool) {
        _approve(msg.sender, spender, value);
        return true;
    }

    function transferFrom(address sender, address recipient, uint256 amount) public chkFrozen(msg.sender) chkLocked(sender, amount) returns (bool) {
        _transfer(sender, recipient, amount);
        _approve(sender, msg.sender, _allowances[sender][msg.sender].sub(amount));
        return true;
    }

    function increaseAllowance(address spender, uint256 addedValue) public returns (bool) {
        _approve(msg.sender, spender, _allowances[msg.sender][spender].add(addedValue));
        return true;
    }

    function decreaseAllowance(address spender, uint256 subtractedValue) public returns (bool) {
        _approve(msg.sender, spender, _allowances[msg.sender][spender].sub(subtractedValue));
        return true;
    }

    function _transfer(address sender, address recipient, uint256 amount) internal {
        require(sender != address(0), "KIP7: transfer from the zero address");
        require(recipient != address(0), "KIP7: transfer to the zero address");
	require(_balances[sender] >= amount && _balances[recipient] + amount >= _balances[recipient]);
        _balances[sender] = _balances[sender].sub(amount);
        _balances[recipient] = _balances[recipient].add(amount);
        emit Transfer(sender, recipient, amount);
    }

    function _mint(address account, uint256 amount) internal {
        require(account != address(0), "KIP7: mint to the zero address");
        _totalSupply = _totalSupply.add(amount);
        _balances[account] = _balances[account].add(amount);
        emit Transfer(address(0), account, amount);
    }

    function _burn(address account, uint256 value) internal {
        require(account != address(0), "KIP7: burn from the zero address");
        _balances[account] = _balances[account].sub(value);
        _totalSupply = _totalSupply.sub(value);
        emit Transfer(account, address(0), value);
    }

    function _approve(address owner, address spender, uint256 value) internal {
        require(owner != address(0), "KIP7: approve from the zero address");
        require(spender != address(0), "KIP7: approve to the zero address");
        _allowances[owner][spender] = value;
        emit Approval(owner, spender, value);
    }

    function _burnFrom(address account, uint256 amount) internal {
        _burn(account, amount);
        _approve(account, msg.sender, _allowances[account][msg.sender].sub(amount));
    }

    function freeze(address target) external onlyOwner returns (bool) {
        _frozen[target] = true;
        emit Freeze(target);
        return true;
    }

    function unFreeze(address target) external onlyOwner returns (bool){
        _frozen[target] = false;
        emit Unfreeze(target);
        return true;
    }

    function isFrozen(address target) external view returns (bool){
        return _frozen[target];
    }
    
    function _lock(address account, uint256 value_, uint256 releaseTime) onlyOwner internal returns (bool) {
	require(releaseTime > now, "lock: Cannot set due to past");
        require(
            _balances[account] >= value_.add(totalLock[account]), "lock: locked total should be smaller than balance"
        );
        totalLock[account] = totalLock[account].add(value_);
        lockbox_arr[account].push(lock_box(value_, releaseTime));

	emit Lock(account, value_, releaseTime);
	return true;
    }
    
    function _unlock(address account, uint256 idx) onlyOwner internal returns(bool){
    	require(lockbox_arr[account].length > 0, "unlock: account does not exist.");
        require(lockbox_arr[account].length > idx, "unlock: idx does not exist.");
        require(lockbox_arr[account][idx].releaseTime < now, "unlock: cannot unlock before releaseTime");

        lock_box storage lock = lockbox_arr[account][idx];
        totalLock[account] = totalLock[account].sub(lock.value_);
        
        lockbox_arr[account][idx] = lockbox_arr[account][lockbox_arr[account].length - 1];
        lockbox_arr[account].pop();

	emit Unlock(account, lock.value_);
        return true;
    }

    function unlockAll(address from) external returns (bool) {
        for (uint256 i = 0; i < lockbox_arr[from].length; i++) {
            if (lockbox_arr[from][i].releaseTime < now) {
                if (_unlock(from, i)) {
                    i--;
                }
            }
        }
        return true;
    }

    function releaseLock(address from) external onlyOwner returns (bool){
        for (uint256 i = 0; i < lockbox_arr[from].length; i++) {
            if (_unlock(from, i)) {
                i--;
            }
        }
        return true;
    }

    function transferWithLockUp(address recipient, uint256 amount, uint256 due) external onlyOwner returns (bool){
        require(
            recipient != address(0),
            "Lock.transferWithLockUp: Cannot send to zero address"
        );
        _transfer(msg.sender, recipient, amount);
        _lock(recipient, amount, due);
        return true;
    }

    function transferFromWithLockUp(address sender, address recipient, uint256 amount, uint256 due) external onlyOwner returns (bool){
        require(
            recipient != address(0),
            "Lock.transferFromWithLockUp: Cannot send to zero address"
        );
        _transfer(sender, recipient, amount);
        _lock(recipient, amount, due);
        return true;
    }

    function listLockInfo(address locked) external view returns (uint256[] memory, uint256[] memory){
        uint256[] memory amounts = new uint256[](lockbox_arr[locked].length);
        uint256[] memory dues = new uint256[](lockbox_arr[locked].length);

        for (uint i = 0; i < lockbox_arr[locked].length; i++) {
            lock_box storage lock = lockbox_arr[locked][i];
            amounts[i] = lock.value_;
            dues[i] = lock.releaseTime;
        }

        return (amounts, dues);
    }

    function totalLocked(address locked) external view returns (uint256 amount, uint256 length) {
        amount = totalLock[locked];
        length = lockbox_arr[locked].length;
    }
    
    function transferOwnership(address _newOwner) public onlyOwner {
        _owner = _newOwner;
    }
}