# Universal Crypto Asset Token(UCAT)

# Simple Summary
  A standard interface for cross chain non-fungible tokens(NFT).

# Abstract
  The following standard allows the implementation of a standard API for NFTs within EOS/IOST/ETH smart contracts. This standard provides basic functionality to track and transfer NFTs.

  NFTs can represent ownership over digital assets.

# Motivation
  A standard interface allows wallet/broker/auction applications to work with any NFT on EOS/IOST/ETH blockchain. A simple EOS/IOST smart contract is provided.

# Specification
The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119.

```
#include <eosiolib/asset.hpp>
#include <eosiolib/contract.hpp>
#include <eosiolib/currency.hpp>
#include <eosiolib/eosio.hpp>
#include <eosiolib/types.hpp>
#include <string>

#define max(a,b) (a>b?a:b)

#define ASSET_EOS_SYMBOL S(4, EOS)
#define EOS_ACCOUNT N(eosio.token)
#define ASSET_XPC_SYMBOL S(4, XPC)
#define XPC_ACCOUNT N(xpctokencore)
#define LOG_EXPIRE_SECOND 3600

#define SYSPARAM_TOKEN_COUNT        1
#define SYSPARAM_MIN_LOG_ID         2
#define SYSPARAM_MAX_LOG_ID         3
#define SYSPARAM_ADMIN_ACCOUNT      4
#define API_URL			    5


class eosnft : public eosio::contract {
public:
    eosnft(account_name self)
        : contract(self),
        sysparams(_self, _self),
        logs(_self, _self),
        tokens(_self, _self),
        accounts(_self, _self)
    {
    }

    /**
     **/
    void apply(account_name contract, uint64_t action);

private:

    // @abi table sysparam i64
    struct sysparam {
        uint64_t id;
        std::string value;

        sysparam() = default;

        EOSLIB_SERIALIZE(sysparam, (id)(value))

        uint64_t primary_key() const { return id; }

    };
    static constexpr uint64_t SYSPARAM_TABLE_NAME = N(sysparam);
    using sysparam_list = eosio::multi_index<SYSPARAM_TABLE_NAME, sysparam>;

    
    // @abi table token i64
    struct token {
        uint64_t id;
        std::string name;
	    account_name owner;
	    std::string imageUrl;
	    std::string category;
	    std::string meta;

        token() = default;
        
        EOSLIB_SERIALIZE(token, (id)(name)(owner)(imageUrl)(category)(meta))
       
        uint64_t primary_key() const { return id; }
    };
    static constexpr uint64_t TOKEN_TABLE_NAME = N(token);
    using token_list = eosio::multi_index<TOKEN_TABLE_NAME, token>;

 
    // @abi table accounts i64
    struct account {
        account_name id;

        account() = default;

        uint64_t tokenCount;

        std::string info;

        EOSLIB_SERIALIZE(account, (id)(tokenCount)(info))

        account_name primary_key() const { return id; }
    };
    static constexpr account_name ACCOUNT_TABLE_NAME = N(account);
    using account_list = eosio::multi_index<ACCOUNT_TABLE_NAME, account>;

    // @abi table log i64
    struct log {
        uint64_t id;
	account_name from;
	account_name to;
	std::string memo;
	uint64_t acttime;

        log() = default;

        EOSLIB_SERIALIZE(log, (id)(from)(to)(memo)(acttime))

        uint64_t primary_key() const { return id; }
    };
    static constexpr uint64_t LOG_TABLE_NAME = N(log);
    using log_list = eosio::multi_index<LOG_TABLE_NAME, log>;

 private:
    account_list accounts;
    token_list tokens;
    log_list logs;
    sysparam_list sysparams;
    
 private:
     account_name get_admin() const {
         const std::string adminAccount = getsysparam(SYSPARAM_ADMIN_ACCOUNT);
         if (adminAccount.empty()) {
             return _self;
         }
         else {
             return eosio::string_to_name(adminAccount.c_str());
         }
     }

     void require_auth_admin() const { eosio::require_auth(get_admin()); }
     
     inline std::string getsysparam(const uint64_t& key) const {
       auto iter = sysparams.find(key);
       if(iter == sysparams.end()){
	        return std::string("");
       }else{
	        return iter->value;
       }
    }

    
    inline void setsysparam(const uint64_t& key, const std::string& val){
	    auto iter = sysparams.find(key);
	    if(iter == sysparams.end()){
		    sysparams.emplace(_self, [&](auto& p) {
		        p.id = key;
		        p.value=val;
		    });
	    }else{
		    sysparams.modify(sysparams.begin(), _self, [&](auto& p) {
			    p.value = val;
		    });
	    }
    }

    inline void addaccounttoken(const account_name& user) {
        auto iter = accounts.find(user);
        if (iter == accounts.end()) {
            accounts.emplace(_self, [&](auto& p) {
                p.id = user;
                p.tokenCount = 1;
                });
        }
        else {
            accounts.modify(accounts.begin(), _self, [&](auto& p) {
                p.tokenCount += 1;
            });
        }
    }

    inline void subaccounttoken(const account_name& user) {
        auto iter = accounts.find(user);
        if (iter == accounts.end()) {
            accounts.emplace(_self, [&](auto& p) {
                p.id = user;
                p.tokenCount = 0;
                });
        }
        else {
            accounts.modify(accounts.begin(), _self, [&](auto& p) {
                if (p.tokenCount > 1){
                    p.tokenCount -= 1;
                }else{
                    p.tokenCount = 0;
                }
           });
        }
    }
    
    inline uint64_t toInt(const std::string& str) {
        if (str.empty()) {
            return 0;
        }
        else {
            std::string::size_type sz = 0;
            return std::stoull(str, &sz, 0);
        }
    }


    inline log getlog(const uint64_t id) const {
        auto iter = logs.find(id);
        return *iter;
    }
    
    inline uint64_t getminlogid() {
        return toInt(getsysparam(SYSPARAM_MIN_LOG_ID));
    }
    
    inline uint64_t getmaxlogid() {
        return toInt(getsysparam(SYSPARAM_MIN_LOG_ID));
    }

    inline void setminlogid(const uint64_t id) {
        setsysparam(SYSPARAM_MIN_LOG_ID, std::to_string(id));
    }

    inline void setmaxlogid(const uint64_t id) {
        setsysparam(SYSPARAM_MAX_LOG_ID, std::to_string(id));
    }

    void logoperator(const uint64_t& id, const account_name& oldowner, const account_name& newowner, const std::string& opcode);

    void clearlog();


 public:
 
    // @abi action
    void assign(const uint64_t id, const account_name newowner);
    
    // @abi action
    void create(const uint64_t id, const std::string category, const std::string name, const std::string imageUrl, const std::string meta);

    // @abi action
    void transfer(const uint64_t id, const account_name newowner);

    // @abi action
    void update(const uint64_t id, const std::string name, const std::string category, const std::string imageUrl, const std::string meta);

};

```
