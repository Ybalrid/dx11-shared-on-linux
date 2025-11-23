# dx11-shared-on-linux

This is just a platform for experimentation. What I want to check is the feasability of any of this.

I want to be able to have two windows applications running in the same wine bottle to be able to share one Direct3D texture.

This is a mechanism that is doable on Winodws, using DXGI shared resources.

Currently this is not functional *at all*. This is just a repo that point to forks of Valve's version of Wine (based on Proton 10) and the current master branch of DXVK.

## Concept

DXVK implements the D3D11 and DXGI APIs (among others) on top of Vulkan.

Currently this library support the calls to GetSharedHandle() and OpenSharedResource() in DirectX 11, but this is apparently only within one process.

In *theory*, you can do the type of sharing I am interested to do entirely in Vulkan, using the External Memory FD extensions.

## Design

Things that must happen:

 - When a DirectX 11 application create a texture with the `D3D11_RESOURCE_MISC_FLAG` bits for `D3D11_RESOURCE_MISC_SHARED` (and maybe `_NTHANDLE` too), the underlying Vulkan image must be allocated in a way where it will be "exportable"
 - When from this DirectX 11 texture, is requested to get it's shareable handle (`ID3D11Texture2D->QuerryInterface(&pID3D11Resource)` then `pID3D11Resource()->GetSharedHandle(&hanle)`), DXVK should be able to do the following:
  - export the file descriptor `fd` for the image, create a arbitrary `HANDLE`  that should be able to uniquely identify this fd
  - effectively send the `fd` to `wineserver`, which effectively is another Linux process running on the machine that represent the Windows NT kernel

 - *The handle is obtained by the remote process using means that are exterior to this sharing scheme*

 - When a DirectX 11 application try to obrain a texture from a shared handle with `ID3D11Device->OpenSharedHandle()`, DXVK should obtain the (duplicated) fd from `wineserver`
 - It should use a Vulkan Dedicated Allocation to import the Vulkan Image from the file descriptor recieved from `wineserver`  that correspond to the obtained handle

None of this is currently implemented in either DXVK or Wine

## Requirements

The Vulkan Device and Instance used by DXVK should

 - Have at least the following Vulkan extensions enabled
   - `VK_KHR_EXTERNAL_MEMORY_CAPABILITIES`
   - `VK_KHR_EXTERNAL_MEMORY_FD`
   - `VK_KHR_DEDICATED_ALLOCATION`
 - Have all processes use the same `VkPhysicalDevice`

(All of these extensions are supported under all linux desktop vulkan drivers that I am aware of)

The wine side should processes should

- Run in the same wine prefix (the same "bottle")
- WineSever should support a way to register and communicate shared vulkan object handles between the processes


