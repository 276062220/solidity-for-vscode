<script lang="ts">
    import { displayEtherValue } from '../../shared/ether';
    import { parseComplexNumber } from '../../shared/validate';
    import ClickableSpan from '../components/ClickableSpan.svelte';
    import InputIssueIndicator from '../components/InputIssueIndicator.svelte';

    import CopyableSpan from '../components/CopyableSpan.svelte';
    import WarningIcon from '../components/icons/WarningIcon.svelte';
    import InfoTooltip from '../components/InfoTooltip.svelte';
    import { getInputFromTopBar, setBalance, showErrorMessage } from '../helpers/api';
    import {
        accounts,
        selectedAccount,
        selectedAccountId,
        selectedValue,
        selectedValueString,
        selectedValueShow,
        setSelectedAccount,
        txParametersExpanded
    } from '../helpers/stores';
    import { getAddress } from 'ethers';

    let selValueUnit = 'ether';

    /**
     * 格式化地址，保留前后指定长度字符，中间用 ... 替换
     * @param address 以太坊地址（如 0x1234...）
     * @param prefixLen 保留的前缀长度（默认 5）
     * @param suffixLen 保留的后缀长度（默认 5）
     * @returns 格式化后的地址字符串
     */
    function formatAddress(
        address: string,
        prefixLen: number = 5,
        suffixLen: number = 5
    ): string {
        if (!address) return '';
        // 如果地址长度小于等于前后保留长度之和，直接返回原地址
        if (address.length <= prefixLen + suffixLen) {
            return address;
        }
        const prefix = address.slice(0, prefixLen);
        const suffix = address.slice(-suffixLen);
        return `${prefix}...${suffix}`;
    }

    function handleAccountChange(event: any) {
        const _selectedAccountIndex = event.detail.value;

        if (
            _selectedAccountIndex === undefined ||
            _selectedAccountIndex < 0 ||
            _selectedAccountIndex >= $accounts.length
        ) {
            setSelectedAccount(null);
            return;
        }

        setSelectedAccount(_selectedAccountIndex);
    }

    function handleValueChange(event: any) {
        let _value = event.target.value;
        if (_value) {
            _value = _value + ' ' + selValueUnit;
        }
        selectedValueString.set(_value ?? null);
        console.log('handleValueChange', $selectedValue, $selectedValueString, $selectedValueShow);
    }

    function handleValueUnitChange(event: any) {
        selValueUnit = event.target.value;
        console.log('handleValueUnitChange', selValueUnit);
    }

    async function topUp() {
        if ($selectedAccount === undefined) {
            return;
        }

        const topUpValue = await getInputFromTopBar(
            $selectedAccount!.balance?.toString(),
            'Update balance of account'
        );
        if (topUpValue === undefined || topUpValue.value === undefined) {
            return;
        }

        let parsedTopUpValue;
        try {
            parsedTopUpValue = parseComplexNumber(topUpValue.value);
        } catch (e) {
            const errorMessage = typeof e === 'string' ? e : (e as Error).message;
            showErrorMessage('Value could not be parsed: ' + errorMessage, true);
            return;
        }

        // convert to number
        const topUpValueNumber = Number(parsedTopUpValue);

        setBalance($selectedAccount!.address, topUpValueNumber);
    }
</script>

{#if accounts !== undefined}
    {#if $txParametersExpanded}
        <section class="flex flex-col gap-1 p-3">
            <div>
                <span class="text-sm">Sender Account</span>
                <vscode-dropdown
                    position="below"
                    class="w-full mb-2"
                    on:change={handleAccountChange}
                >
                    <span slot="label">Sender Account</span>
                    {#each $accounts as account, i}
                        <vscode-option
                            value={i}
                            selected={account.address == $selectedAccount?.address}
                            >{i} {formatAddress(getAddress(account.address))} ({displayEtherValue(BigInt(account.balance))})</vscode-option
                        >
                    {/each}
                    <!-- @dev hack to display selected account -->
                    <span slot="selected-value">
                        {#if $selectedAccountId !== null}
                            Account {$selectedAccountId} {displayEtherValue(BigInt($selectedAccount?.balance ?? 0))}
                        {/if}
                    </span>
                </vscode-dropdown>

                {#if $selectedAccount !== null}
                    <div class="w-full px-1 mb-3">
                        <div class="w-full flex flex-row gap-1 items-center h-[20px]">
                            <!-- <span class="flex-1 truncate text-sm">{$selectedAccount.address}</span> -->
                            <!-- <span class="flex-1 truncate text-sm">{accounts[selectedAccountIndex].address}</span> -->
                            <CopyableSpan
                                text={getAddress($selectedAccount.address)}
                                className="flex-1 truncate text-sm"
                            />
                            <!-- <CopyButton callback={() => copyToClipboard($selectedAccount.address)} /> -->
                        </div>
                        <div class="w-full flex flex-row gap-1 items-center h-[20px]">
                            <ClickableSpan className="text-sm flex-1" callback={topUp}>
                                {displayEtherValue(BigInt($selectedAccount.balance))}
                            </ClickableSpan>
                            <!-- <span class="text-sm flex-1">{accounts[selectedAccountIndex].balance}ETH</span> -->
                            <!-- <IconButton callback={topUp}>+</IconButton> -->
                        </div>
                    </div>
                {/if}
            </div>

            <div class="w-full">
                <div class="flex flex-row gap-1">
                    <span class="text-sm">Transaction Value</span>
                    <InfoTooltip
                        content="The ether value which will be sent with the transaction."
                    />
                </div>
                <div>
                <div class="flex flex-row">
                    <div style="flex: 1;">                        
                        <vscode-text-field
                            placeholder="0 ETH"
                            class="w-full"
                            value={$selectedValueShow}
                            on:change={handleValueChange}
                        >
                            <div slot="end" class="flex items-center">
                                {#if $selectedValue === null}
                                    <InputIssueIndicator type="danger">
                                        <span class="text-sm">Value could not be parsed</span>
                                    </InputIssueIndicator>
                                {:else}
                                    <span slot="end" class="flex justify-center align-middle leading-5"
                                        >Ξ</span
                                    >
                                {/if}
                            </div>
                        </vscode-text-field>
                    </div>
                    <div style="padding-left: 3px;">
                        <vscode-dropdown
                            id="valueUnit"
                            position="below"
                            class="w-full mb-2"
                            on:change={handleValueUnitChange}
                        >
                            <vscode-option value='wei'>wei</vscode-option>
                            <vscode-option value='gwei'>gwei</vscode-option>
                            <vscode-option value='finney'>finney</vscode-option>
                            <vscode-option value='ether' selected='true'>ether</vscode-option>
                        </vscode-dropdown>
                    </div>
                </div>
                </div>
            </div>
        </section>
    {/if}
{:else}
    <div class="flex flex-row gap-1 items-center p-3">
        <WarningIcon />
        <span class="text-vscodeForegroundSecondary font-normal">No accounts found</span>
    </div>
{/if}
